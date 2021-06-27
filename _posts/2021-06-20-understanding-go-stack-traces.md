---
layout: post
title: Understanding Go Stack Traces(what you see when panic happens)
categories: Go
---

```go
panic: runtime error: index out of range [0] with length 0

goroutine 1 [running]:
main.max(0xc000034778, 0x0, 0x0, 0xc000062058)
  /tmp/sandbox466854008/prog.go:8 +0x66
main.main()
  /tmp/sandbox466854008/prog.go:4 +0x33
```

There are a lot of ways in which your program can panic - runtime errors like index out of range, nil pointer dereference or you simply choose to call `panic`. If you have spent some time writing Go program, you should be pretty familiar with messages like the one above - you get one like that every time your program panic. Other than the first line, which shows the reason of panic/anything you pass to `panic`, the rest of the message is a **stack trace** of the goroutine that panicked, which is going to be the topic of this post.

Go stack traces are not intuitive to read. Admittedly, I had absolutely no idea what those hex number meant and how to interprete the first time I saw them(and quite long after that). But before we go into the details of stack traces, what does the "stack" mean in the first place?

## Call stack

> Feel free to skip this section if you already know what call stack is

The "stack" in "stack trace" means "call stack". It is basically the old FIFO stack data structure, just that it is used to store information about the active function calls of your program. Each goroutine in your Go code is started with a function call, and that function can contain function calls:

```go
package main

func main() {
  // before
  someFn()
  // after
}

func someFn() {
  // do something
  return
}
```

When your goroutine starts, the call stack only has one item - the "stack frame" of the starting function call. In the example above, the main goroutine starts with only the stack frame of the `main` function in the call stack.

```go
package main

func main() {
  // before <---------
  someFn()
  // after
}

func someFn() {
  // do something
  return
}
```

```
   +--------------------+
   |                    |
   |                    |
   |                    |
   |                    |
   |                    |
   |                    |
   |                    |
   ----------------------
   | main's stack frame |
   +--------------------+
      main goroutine's
      call stack
```

Each stack frame corresponds to a function call that has not returned yet, and it is a data structure that stores information about the function call, such as:

- what is the function being called
- the arguments
- local variables of the function
- where to return to after the function call (return address)
- where is the goroutine executing in the function (program counter)

When a new function call happens, the new function call becomes active, but the current function is still active, because the goroutine will return to it after the new function call finishes. Therefore, the stack frame of the new function call gets pushed to the call stack.

```go
package main

func main() {
  // before
  someFn() // <---------
  // after
}

func someFn() {
  // do something <---------
  return
}
```

```
   +--------------------+
   |                    |
   |                    |
   |                    |
   |                    |
   |      (pushed)      |
   ----------------------
   |someFn's stack frame|
   ----------------------
   | main's stack frame |
   +--------------------+
      main goroutine's
      call stack
```

After a function call returns, it stops being active, so its stack frame gets poped off the call stack:

```go
package main

func main() {
  // before
  someFn()
  // after <---------
}

func someFn() {
  // do something
  return
}
```

```
   +--------------------+
   |                    |
   |                    |
   |                    |
   |                    |
   |                    |
   |                    |
   |      (poped)       |
   ----------------------
   | main's stack frame |
   +--------------------+
      main goroutine's
      call stack
```

Call stacks allows programs to keep track of the local context of function calls. It's not hard to imagine how information in the stack frames can come in handy in the postmortem after your program panics, and that's why there are stack traces and why they are printed when panic happens. A stack traces, like the one shown at the beginning, is a report of the call stack of a certain goroutine, or more generally a certain thread of control, at a certain place in the program.

Now that we understand what call stack is and what stack traces are, we are going to break down some Go stack traces, and show you exactly what each part means.

## Components of stack traces

To get a stack trace to start with, let's use this program:

```go
package main

func main() {
	panicIfNonZero(1)
}

//go:noinline - ignore this line first
func panicIfNonZero(x int) {
	if x == 0 {
		return
	}
	panic(x)
}
```

The program calls a function that panics if the input is non-zero with a non-zero input, so it panics:

```
panic: 1

goroutine 1 [running]:
main.panicIfNonZero(0x1)
	/tmp/sandbox180829046/prog.go:12 +0x4a
main.main()
	/tmp/sandbox180829046/prog.go:4 +0x2a
```

It is not hard to see that the stack trace has 2 levels, the first one starting with`main.panicIfNonZero`, another starting with `main.main`. It reports the call stack when the panic happened in `main.panicIfNonZero`, which is called by `main.main`, and each of the level corresponds to a stack frame. Let me give each part a name:

```
            +--------------------------------- goid of the goroutine
            | +-------+----------------------- goroutine status
            | -       -
  goroutine 1 [running]:

  +-----------------+------------------------- function name
  |                 | +-+--------------------- arguments
  |                 | - -
  main.panicIfNonZero(0x1)
    +---------------------------+------------- file name
    |                           | +----------- line number
    |                           | |  +---+---- relative program counter
    |                           | |  -   -
    /tmp/sandbox180829046/prog.go:12 +0x4a

 +-------------------------------------------- another level
  main.main()                               |
    /tmp/sandbox180829046/prog.go:4 +0x2a   |
 +------------------------------------------+
```

The header is straightforward. It just tells you in which goroutine's stack trace is that, and what's the state of the goroutine.

In the first level, the function name, file name and line number need no explaination, but what does the program counter and function argument list mean?

### Relative program counter

The relative program coutner in hexadecimal number tells you where your program exactly is inside the function. The hexadecimal number represents the relative position of the current instruction from the beginning of the function. To see what this means, let's print out the assembly listing of the sample program with `go tool compile -S main.go`. Here I only show the assembly code for the 2 functions

```
"".main STEXT size=59 args=0x0 locals=0x10
        0x0000 00000 (main.go:3)        TEXT    "".main(SB), ABIInternal, $16-0
        0x0000 00000 (main.go:3)        MOVQ    (TLS), CX
        0x0009 00009 (main.go:3)        CMPQ    SP, 16(CX)
        0x000d 00013 (main.go:3)        PCDATA  $0, $-2
        0x000d 00013 (main.go:3)        JLS     52
        0x000f 00015 (main.go:3)        PCDATA  $0, $-1
        0x000f 00015 (main.go:3)        SUBQ    $16, SP
        0x0013 00019 (main.go:3)        MOVQ    BP, 8(SP)
        0x0018 00024 (main.go:3)        LEAQ    8(SP), BP
        0x001d 00029 (main.go:3)        PCDATA  $0, $-2
        0x001d 00029 (main.go:3)        PCDATA  $1, $-2
        0x001d 00029 (main.go:3)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:3)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:3)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:4)        PCDATA  $0, $0
        0x001d 00029 (main.go:4)        PCDATA  $1, $0
        0x001d 00029 (main.go:4)        MOVQ    $1, (SP)
        0x0025 00037 (main.go:4)        CALL    "".panicIfNonZero(SB)
        0x002a 00042 (main.go:5)        MOVQ    8(SP), BP
        0x002f 00047 (main.go:5)        ADDQ    $16, SP
        0x0033 00051 (main.go:5)        RET
        0x0034 00052 (main.go:5)        NOP
        0x0034 00052 (main.go:3)        PCDATA  $1, $-1
        0x0034 00052 (main.go:3)        PCDATA  $0, $-2
        0x0034 00052 (main.go:3)        CALL    runtime.morestack_noctxt(SB)
        0x0039 00057 (main.go:3)        PCDATA  $0, $-1
        0x0039 00057 (main.go:3)        JMP     0

"".panicIfNonZero STEXT size=82 args=0x8 locals=0x18
        0x0000 00000 (main.go:8)        TEXT    "".panicIfNonZero(SB), ABIInternal, $24-8
        0x0000 00000 (main.go:8)        MOVQ    (TLS), CX
        0x0009 00009 (main.go:8)        CMPQ    SP, 16(CX)
        0x000d 00013 (main.go:8)        PCDATA  $0, $-2
        0x000d 00013 (main.go:8)        JLS     75
        0x000f 00015 (main.go:8)        PCDATA  $0, $-1
        0x000f 00015 (main.go:8)        SUBQ    $24, SP
        0x0013 00019 (main.go:8)        MOVQ    BP, 16(SP)
        0x0018 00024 (main.go:8)        LEAQ    16(SP), BP
        0x001d 00029 (main.go:8)        PCDATA  $0, $-2
        0x001d 00029 (main.go:8)        PCDATA  $1, $-2
        0x001d 00029 (main.go:8)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $2, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
        0x001d 00029 (main.go:9)        PCDATA  $0, $0
        0x001d 00029 (main.go:9)        PCDATA  $1, $0
        0x001d 00029 (main.go:9)        MOVQ    "".x+32(SP), AX
        0x0022 00034 (main.go:9)        TESTQ   AX, AX
        0x0025 00037 (main.go:9)        JNE     49
        0x0027 00039 (main.go:10)       PCDATA  $0, $-1
        0x0027 00039 (main.go:10)       PCDATA  $1, $-1
        0x0027 00039 (main.go:10)       MOVQ    16(SP), BP
        0x002c 00044 (main.go:10)       ADDQ    $24, SP
        0x0030 00048 (main.go:10)       RET
        0x0031 00049 (main.go:12)       PCDATA  $0, $0
        0x0031 00049 (main.go:12)       PCDATA  $1, $0
        0x0031 00049 (main.go:12)       MOVQ    AX, (SP)
        0x0035 00053 (main.go:12)       CALL    runtime.convT64(SB)
        0x003a 00058 (main.go:12)       PCDATA  $0, $1
        0x003a 00058 (main.go:12)       LEAQ    type.int(SB), AX
        0x0041 00065 (main.go:12)       PCDATA  $0, $0
        0x0041 00065 (main.go:12)       MOVQ    AX, (SP)
        0x0045 00069 (main.go:12)       CALL    runtime.gopanic(SB)
        0x004a 00074 (main.go:12)       XCHGL   AX, AX
        0x004b 00075 (main.go:12)       NOP
        0x004b 00075 (main.go:8)        PCDATA  $1, $-1
        0x004b 00075 (main.go:8)        PCDATA  $0, $-2
        0x004b 00075 (main.go:8)        CALL    runtime.morestack_noctxt(SB)
        0x0050 00080 (main.go:8)        PCDATA  $0, $-1
        0x0050 00080 (main.go:8)        JMP     0
```

The first 3 columns shows the relative position of each line in hexadecimal and decimal, and the line position in the source file. If we find `0x2a` in `main`, and in `0x4a` in `panicIfNonZero`,

```
        0x0025 00037 (main.go:4)        CALL    "".panicIfNonZero(SB)
        0x002a 00042 (main.go:5)        MOVQ    8(SP), BP <-------
        0x002f 00047 (main.go:5)        ADDQ    $16, SP

        0x0045 00069 (main.go:12)       CALL    runtime.gopanic(SB)
        0x004a 00074 (main.go:12)       XCHGL   AX, AX <-------
        0x004b 00075 (main.go:12)       NOP
```

we find that the relative program counter value points to the location of the instruction following the function call.

### Arguments

In the example above, the only argument that is passed is the `int` value `1` to `panicIfNonZero`, and because that we have `main.panicIfNonZero(0x1)`. `0x1` is just `1` in hexadecimal, which seems pretty straight forward. Well, let's try another example:

```go
package main

func first(s string) byte {
	return s[0]
}

func main() {
	first("")
}
```

Here we attemp to sum over an empty slice of `int` using the `sum` that expects at least 1 element in the input `[]int`. The output looks like this:

```
panic: runtime error: index out of range [0] with length 0

goroutine 1 [running]:
main.sum(0xc000034778, 0x0, 0x0, 0xc000062058)
	/tmp/sandbox977806873/prog.go:4 +0x65
main.main()
	/tmp/sandbox977806873/prog.go:12 +0x33
```

As expected, we get a panic, but why are there 4 items in the arguments when there is only one arguement passed to the function? Well, this has to do with how values of different types are encoded in Go.

## Encoding of function arguments in stack trace

Actually, each item in the list doesn't correspond to a function argument. Instead, they represent a "word", which in most cases means 4 or 8 bytes, depending on whether your computer's CPU is a 32-bit one or a 64-bit one. The arguments passed to

**Why are there 4 values for one slice????**
