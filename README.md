# Ultimate Go

https://github.com/ardanlabs/gotraining/blob/master/topics/courses/go/README.md

## Disclaimer

All the information below are **my personal notes** from attending William Kennedy's adapted Ultimate Go (1 day length) workshop at Gophercon EU 2018 in Iceland. Some of the information might be imprecise and even wrong, so make sure to fact check what's written here and push any changes as you see fit.

These are rough notes and do not reflect the quality and content of the workshop that was delivered.

I highly encourage anyone that stumbles upon this repo to take the full [Ultimate Go](https://www.ardanlabs.com/ultimate-go).

## Table of contents

- [Ultimate Go](#ultimate-go)
    - [Disclaimer](#disclaimer)
    - [Table of contents](#table-of-contents)
    - [Performance Focus factors](#performance-focus-factors)
    - [Language Mechanics](#language-mechanics)
        - [Memory structure](#memory-structure)
        - [Goroutines (Coroutines really, G is for Go)](#goroutines-coroutines-really--g-is-for-go)
    - [Value semantics](#value-semantics)
        - [Pros](#pros)
        - [Cons](#cons)
    - [Pointer semantics](#pointer-semantics)
        - [Pros](#pros)
        - [Cons](#cons)
        - [Mechanics](#mechanics)
            - [Good](#good)
            - [Bad](#bad)
    - [Moar about the stack](#moar-about-the-stack)
    - [HEAP](#heap)
    - [Data structures](#data-structures)
    - [Memory structure](#memory-structure)
            - [Example of multiple runs](#example-of-multiple-runs)
    - [Semantics](#semantics)
    - [Decoupling](#decoupling)
- [Concurrency](#concurrency)
    - [Types of work by threads](#types-of-work-by-threads)
    - [Switching from one thread to another](#switching-from-one-thread-to-another)
    - [Go and Scheduling / Concurrency](#go-and-scheduling---concurrency)

## Performance Focus factors

1. Latency:
    * I/O.
    * Network.
    * Timeouts.

2. Garbage collection
    * Generate the right type of Garbage.

3. Accessing Data

4. Algorithm efficiency
    * Machines are already very powerful, we can be less optimized today.

## Language Mechanics

* [Pointers](https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/pointers/README.md)


### Memory structure

Data Segment
Stacks
Heap

### Goroutines (Coroutines really, G is for Go)

See: https://play.golang.org/p/JJMHWiZ9h9

Glossary:
    * `AF`: Active Frame.
    * `Semantics`: Behaviour.
    * `Mechanics`: How it works.

* Have their own resizeable stack, starting at 2K.
    * Segmentation for safety.

* Calling (executing) a function moves the AF to nother part of the stack.


Any memory below the active frame has no integrity and is stale.

// TODO: Diagram stack.

Pointers are a special variable type 4 to 8 bytes depending on the architecture.
`func increment(inc *int)` is still a pass by value, the value is the pointer address.

## Value semantics

### Pros
    * Immutability.
    * No side effects.
    * Allocations happen on the stack (Free, sorta).

### Cons
    * Efficiency (Multiple copies of the data on multiple frames).

## Pointer semantics

### Pros
    * Efficiency (Easy to maintain consistency).

### Cons
    * Side effects (Worsened in concurrent programs).
    * Allocations on the Heap.


### Mechanics

https://play.golang.org/p/sRcPykfoi1d

Allocations are only counted for

Call func => Initialize Stack => Finish func => Switch AF => Invalidates stacks below the AF.


Go compiler performs Static code analysis:
    * Escape Analysis (Any memory initialized down the stack is shared up).

Any shared variables upwards on the call stack will be **allocated** in the heap.
Avoid pointer semantics on construction unless you return it directly or construct it on the pass val.


####  Good
```go
func createUserV2() *user {
	u := user{
		name:  "Bill",
		email: "bill@ardanlabs.com",
	}

	println("V2", &u)

	return &u
}
```

####  Bad
```go
func createUserV2() *user {
	u := &user{
		name:  "Bill",
		email: "bill@ardanlabs.com",
	}

	println("V2", u)

	return u
}

// OR

func createUserV2() user {
	u := &user{
		name:  "Bill",
		email: "bill@ardanlabs.com",
	}

	println("V2", u)

	return *u
}
```

**Rule of thumb: Use semantics on return to hint what kind of value you're returning**


**THIS IS SO AWESOME**
Optimizing for performance, use `-gcflags "-m -m"`: will output reasoning behind escape analysis.

Escape analysis is about moving stuff from the **Stack** to the **Heap**.
Value semantics should be the default option because **allocations hurt** (Even tho more performant).

The size of Stacks are set in stone and are known at compile time.
If the compiler does not know the size of something it has to move it to the **heap**.

## Moar about the stack

https://play.golang.org/p/tpDOwBCvqW

Goroutines are awesome cause they have a 2K stack.

They are resizeable (growable).

1. Run out of space in the stack.
2. Create a new (bigger) memory space.

In the example below we can the memory positions of a string variable hello that exists
in the main.main goroutine stack and how the memory addresses change as the stakc is resized.

```
0 0x1044dfa0 HELLO
1 0x1044dfa0 HELLO
2 0x10455fa0 HELLO
3 0x10455fa0 HELLO
4 0x10455fa0 HELLO
5 0x10455fa0 HELLO
6 0x10465fa0 HELLO
7 0x10465fa0 HELLO
8 0x10465fa0 HELLO
9 0x10465fa0 HELLO
```

Thus, it creates new memory addresses to contain the contents of the pointers, thus new pointer addresses.

**IN Go, the stack is isolated to that goroutine, and variables and pointers in the stack are self contained.**

When memory is shared, the variables (Addr) move to the stack (See upwards behaviour).

When deciding to share memory by pointers, make sure that your usecase actually needs it. i.e. Data integrity.

## HEAP

https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/pointers/README.md#garbage-collection

Concurrent, Mark & Sweep, Garbage collector.

1. Value semantics.
2. Pointer semantics.

Go tries to keep resource utilisation at the minimum. Go asks:

*What's the minimum Heap Size I can have so I can run efficiently (at a reasonable pace) below 100 microseconds.*

Collector is allowed to use up to 25% of total CPU capacity on resource contention.

TO stop the world, you've got to stop all of the goroutines and bring them to a safe point the only current mechanism
to do so is stopping them at function calls (up to 1.10).

Theres 2 stop the world points:

1. Turning the Write barrier on.
2. Scan stacks to the Active Frames.
3. Everything starts white.
4. Things move to grey and get marked either White or Black ocne the usage is decided.
5. Once there's no more Grey items, Mark phase is done
6. SWEEP => Reclaim WHITE objects not in use.
7. OFF => GC Disabled.


White, Gray, Black.

once we're done, everything is White or Black.

Black means the _object_ is used.
White means the _object_ is safe to be swept (deleted).

Less is more.

### Example heap distribution

==============  4 MB

  Transient ^

--------------  2 MB

  Permanent

==============


## Data structures

### Arrays

https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/arrays/README.md

https://youtu.be/WDIkqP4JbkE?t=1129 Small is fast.


## Memory structure

Main memory is broken into cache lines.
Cache lines => 64 bytes


Create code that has predictable access patterns to memory.

Prefetches.
Prefetches read your program's access to memory and try to predict the access pattern to memory.

Arrays and slices give us contiguous strides of access to the memory.

Arrays are the most important data structure there is today.

Slice is the most important data structure in Go.

Most of the time the slice is the right data structure.


TLB Cache.

When a program comes up is given a full map of virtual memory.

Memory is managed in Memory Pages.

![](https://drawings.jvns.ca/drawings/malloc.svg)

#### Example of multiple runs

```
Elements in the link list 4194304
Elements in the matrix 4194304
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/caching
BenchmarkLinkListTraverse-8   	    1000	   6767661 ns/op	       0 B/op	       0 allocs/op
BenchmarkLinkListTraverse-8   	    1000	   7207894 ns/op	       0 B/op	       0 allocs/op
BenchmarkLinkListTraverse-8   	     500	   6474441 ns/op	       0 B/op	       0 allocs/op
BenchmarkColumnTraverse-8     	     500	  11459971 ns/op	       0 B/op	       0 allocs/op
BenchmarkColumnTraverse-8     	     500	  11157890 ns/op	       0 B/op	       0 allocs/op
BenchmarkColumnTraverse-8     	     500	  10665148 ns/op	       0 B/op	       0 allocs/op
BenchmarkRowTraverse-8        	    1000	   3959297 ns/op	       0 B/op	       0 allocs/op
BenchmarkRowTraverse-8        	    1000	   4000582 ns/op	       0 B/op	       0 allocs/op
BenchmarkRowTraverse-8        	    1000	   3924701 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/caching	56.891s

```

https://github.com/ardanlabs/gotraining/tree/master/topics/go#data-oriented-design


TODO: Watch:
* [CPU Caches and Why You Care](https://www.youtube.com/watch?v=WDIkqP4JbkE)
* [NUMA Deep Dive](http://frankdenneman.nl/2016/07/06/introduction-2016-numa-deep-dive-series/)
* [Data-Oriented Design and C++](https://www.youtube.com/watch?v=rX0ItVEVjHc)

Read [Data Oriented Design](https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/arrays/README.md#data-oriented-design)


## Semantics

Designing for value or pointer semantics

If the alteration changes the core value of what you're changing you want to return a new value without altering the state.

If the alteration modifies a property but doesn't modify the underlying state or meaning of the data then you want to youse pointer semantics.

**Avoid mixing semantics in different types**

Never go from pointer to value semantics, it causes SHIT TO GO DOWN.


## Decoupling

Make the choice between Performance and decoupling. Interfaces provide double indirection which causes the escape
analysis to move the types that use the interfaces to the heap even if they're not escaping.


# Concurrency

Thread states

* Running (Or executing).
* Runnable (requesting CPU time).
* Waiting (Lot of substatuses).

## Types of work by threads

* CPU Bound work: Thread never goes into waiting state.
* I/O Bound work: Threads go back and forth from Running to Runnable / Waiting.

## Switching from one thread to another

* Context switches: expensive.
* Less is more, less threads, each thread gets more time.
* Non deterministic switches: Cannot make assumptions in the code.

## Go and Scheduling / Concurrency

Go is a cooperative scheduler.
`runtime` does the cooperation.


```go
// Pprofs to stdout, so if you have any info going to standard out already
// Make sure to point it to some other io.Writer.
pprof.StartCPUProfile(os.Stdout)
defer pprof.StopCPUProfile()
```

```shell
$ ./trace > p.out
$ go tool pprof p.out
(pprof) list find
(pprof) web list find
(pprof) top 40 -cum
(pprof) list <function name>
```


### Trace

```go
// Traces out to stdout, so if you have any info going to standard out already
// Make sure to point it to some other io.Writer.
trace.Start(os.Stdout)
defer trace.Stop()
```


```shell
# Opens up
$ go tool trace t.out
```


### Benchmarks

// Add memory profile
```shell
-memprofile p.out
```
