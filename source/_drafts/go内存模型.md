---
title: go内存模型
date:
tags:
---

# Introduction

	Go的内存模型详述了“在一个groutine中对变量进行读操作能够侦测到在其他goroutine中对该变量的写操作”的条件.

# Advice

	如果程序中修改数据时有其他goroutine同时读取，那么必须将操作串行化。
	
	为了串行化访问，尽量使用sync和sync/atomic等方法来保护数据。
	
	如果你一定要阅读文档剩下的部分来理解程序的行为，那你就太聪明了。
	
	不要自作聪明！！!

# Happens Before

### 背景：

	对于一个goroutine来说，它其中变量的读, 写操作执行表现必须和从所写的代码得出的预期是一致的。在不改变程序表现的情况下，编译器和处理器为了优化代码可能会改变变量的操作顺序即: 指令乱序重排。
	
	为了解决这种二义性问题，Go语言中引进一种偏序关系：happens before，它用于描述对内存操作的先后顺序问题。

### 定义：

	假设A和B表示一个多线程的程序执行的两个操作。如果A happens-before B，那么A操作对内存的影响 ，可以被B操作观测到。
	
	A不是happens-before B，且B不是happens-before A，则A和B并发。
	
	传递性： A happens-before B， B happens-before C     =>    A happens-before  C

### 说明：

A read r of a variable `v` is ***allowed*** to observe a write w to `v` if both of the following hold:

1. r does not happen before w.
2. There is no other write w' to `v` that happens after w but before r.

To guarantee that a read r of a variable `v` observes a particular write w to `v`, ensure that w **is the only write** r is allowed to observe. That is, r is *guaranteed* to observe w if both of the following hold:

1. w happens before r.

2. Any other write to the shared variable `v` either happens before w or after r.

第二组条件比第一组条件更严格，因为不允许并发.

将变量v自动初始化为零也是属于这个内存操作模型。

读写超过一个机器字长度的数据，顺序也是不能保证的。

# Synchronization



### Initialization

	Program initialization runs in a single goroutine, but that goroutine may create other goroutines, which run concurrently.
	
	If a package `p` imports package `q`, the completion of `q`'s `init` functions happens before the start of any of `p`'s.
	
	The start of the function `main.main` happens after all `init` functions have finished.

### Goroutine creation

	The `go` statement that starts a new goroutine happens before the goroutine's execution begins.

For example, in this program:

```Go
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

calling `hello` will print `"hello, world"` at some point in the future (perhaps after `hello` has returned).

### Goroutine destruction

The exit of a goroutine is not guaranteed to happen before any event in the program. For example, in this program:

```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

the assignment to `a` is not followed by any synchronization event, so it is not guaranteed to be observed by any other goroutine. In fact, an aggressive compiler might delete the entire `go` statement.

If the effects of a goroutine must be observed by another goroutine, use a synchronization mechanism such as a lock or channel communication to establish a relative ordering.

### Channel communication

	1. A send on a channel happens before the corresponding receive from that channel completes.

This program:

```
var c = make(chan int, 10)   // buffered
var a string

func f() {
	a = "hello, world"
	c <- 0		// <=> close(c)
}

func main() {
	go f()
	<-c
	print(a)
}
```

2. The closing of a channel happens before a receive that returns a zero value because the channel is closed.

3. A receive from an unbuffered channel happens before the send on that channel **completes****.

   ```
   var c = make(chan int) 	// unbuffered
   var a string

   func f() {
   	a = "hello, world"
   	<-c
   }
   func main() {
   	go f()
   	c <- 0
   	print(a)
   }
   ```

4. The *k*th receive on a channel with capacity *C* happens before the *k*+*C*th send from that channel completes.

   通常用来限制最大并发数

   ```
   var limit = make(chan int, 3)

   func main() {
   	for _, w := range work {
   		go func(w func()) {
   			limit <- 1
   			w()
   			<-limit
   		}(w)
   	}
   	select{}
   }
   ```

### Locks

For any variable `l` and *n* < *m*, call *n* of `l.Unlock()` happens before call *m* of `l.Lock()` returns.

```Go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

通常没这么用

### Once

A single call of `f()` from `once.Do(f)` happens (returns) before any call of `once.Do(f)` returns.

```
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

# Incorrect synchronization

```
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

it can happen that `g` prints `2` and then `0`.



Double-checked locking：

```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

This version can (incorrectly) print an empty string instead of `"hello, world"`.



```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```



```
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```



### 参考：https://golang.org/ref/mem

###  继续学习：

##### 			go内存管理： 分配、回收、逃逸分析
