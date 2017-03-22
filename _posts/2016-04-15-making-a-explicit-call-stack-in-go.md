---
title: Making a explicit call stack in Go
date: 2016-04-15 05:03:00
---

_You may want to [jump directly to the core thing](#reifying-the-call-stack), if you already know how stacks work._

### Variables as call stack fields

A few days ago I noted this:

<blockquote class="twitter-tweet" data-lang="en"><p lang="es" dir="ltr">Me rechina lo mucho que se confunde «reasignabilidad» de variables con mutabilidad de valores. Y el var/let de Swift no ayuda.</p>&mdash; Toni Cárdenas (@tcardv) <a href="https://twitter.com/tcardv/status/716969899877408768">April 4, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

(Rough translation: "It's annoying how often variable "reassignability" and value mutability are confused. And Swift's var/left doesn't help.")

What I meant is, for example, in Swift, `let x = y` means that `x` can't be reassigned to another value, but `y` sometimes can change, thus changing `x`'s underlying value.

Now, if we mean variables in the mathematical sense of "names" or "bindings", this sentence really has a point. But, thinking a bit more about it, in a very direct sense, in an imperative language local **variables _are_ part of a value** themselves. This value is **the call stack**, the data structure that links function calls and stores their local data; and a variable is **like a field in this structure**, identifying a bunch of contiguous memory.

Keeping that in mind, Swift's behavior makes sense: `let x = y` means that the piece of contiguous memory at the call stack named by `x` won't change ever. If `y` is a reference to other piece of memory, like, for example, a class instance is, what `y` points to can change without `x`'s memory changing, so `x` can be declared with `let`. But things like structs, arrays, and `UnsafeMutablePointer`, which may need to change the piece of stack memory that `x` represents, require that `let` to be a `var` instead.

**Programming languages let us take the call stack for granted.** While all other values are explicitly managed, the call stack is just there. Many (most?) programmers use it every day without even knowing of its existence. But, of course, that's just an illusion: under the hood, **all there is is cold memory locations, the program counter, and a CPU** stumbling its way thorugh all it.

In order to see what goes on more clearly, let's make an experiment: **let's _not_ use the language's built-in support for the call stack**. Instead, **let's make the call stack ourselves**, setting things up explicitly.

### What is the call stack?

In order to know what to do, let's first recap what the call stack is, and how it operates.

_If you know about this, just jump to [the next section](#reifying-the-call-stack). Or not. A little refresher could be fun._

In a typical program, **memory is split in three categories**: _static_ memory, where the code and global variables live; the _heap_, where things we create at runtime are put; and **the _stack_**, a growing/shrinking contiguous piece of memory **where function calls put their local data**.

Before a function is called, **a new _stack frame_ is put above the previous one**. A stack frame is a piece of memory **with enough space for the function's arguments, the local variables**, and two more "internal" things: the **memory address where the stack frame below begins**, and the memory address of the point at the current function's code just _after_ the call happens. This is the location of **the code that must be executed when the function returns**.

Then, the memory address where this new stack frame memory begins is written to a special CPU register, called **the _stack base_ (SB)**. The function's code refers to local variables as relative to whatever address is stored in the SB register at the time. This way, **the same function's code** can be reused for **all distinct calls** to the same function.

The memory address where the start of the function's code is located is written to another special CPU register, called **the _program counter_ (PC)**. The CPU reads the memory location stored in this register in an endless loop to fetch the next instruction it needs to execute. Thus, by doing this, **we tell the CPU to jump to the invoked function**'s code.

![Diagram of how a function call operates in the stack.](http://b51.imgup.net/stack12cd7.gif)

When the function returns, it **sets SB to its previous value** (ie. to the caller's stack frame) and **sets PC to the return location**, both of which were stored in the stack frame, thus continuing with the caller function's execution.

![Diagram of how a function return operates in the stack.](http://y38.imgup.net/stack2bd92.gif)

I hope you can see why this is called [a stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)): it is a _linked list_ of _stack frames_, linked by their _stack base_ addresses, with a _push_ operation whenever a function is called, and a _pop_ operation whenever a function returns. You've probably encountered it in your day-to-day programming, at least, when something crashes, and you get a _stack trace_ that prints the line numbers corresponding to the current PC, plus to all return addresses currently present in each stack frame.

### Reifying the call stack

So, as we've seen, **a stack trace is just a linked list**. The language programs this for us, but we can do it ourselves, just like with every other data structure.

To avoid using the implicit call stack, **we need to avoid, well, calling functions!** This may be feasible in C and friends, but not so much in most languages, including Go, which is the one we're using. What we're going to do is **set up a tiny "virtual machine" that executes our real code**, in which we will be unable to do certain things. In particular, **we can't:**

* Call functions.
* Return values.
* Use local variables.
* Use arguments.

Instead, we need to simulate those things. But how?

First, we need to make stack frames somehow. Stack frames is where we need to store:

* Local variables, including temporals and parameters.
* The address we should put the function's result value into.
* A link to the previous stack frame (the previous SB).
* The address of the code we should return to (the next PC).

We're going to create **a struct type per function** holding **its stack frame data**.

**Local variables** can be just **plain fields** in this struct, and the **result value's address** can be just a **pointer field**.

For the other two, **we can use a _method value_**. A method value holds both a reference to a value, and to a method implicitly applied to a value. This value will be **the stack frame struct type** (the previous SB), and the method will be **the code which uses the stack frame** (the return address that will be stored in PC).

Let's transform the code from the simple example from above. This is `main`:

```go
func main() int {
	a := 3
	b := 4
	c := add(a, b)
	return c
}
```

Instead of returning nothing, **our VM's `main` returns an int**, which will be interpreted as the **exit code** for the program. We're going to use it to **see the result of our computations**, as, of course, calling `print` or similar is forbidden!

`main`'s stack frame type would look like this:

```go
type mainStackFrame struct {
	// Local variables.
	a int
	b int
	c int

	// Where we're going to set main's return value.
	resultAddress *int
	// What should be called when main returns.
	returnContext func()
}
```

Now, let's convert `add`'s stack frame:

```go
func add(a, b int) int {
	result := a + b
	return result
}
```

This needs:

```go
type addStackFrame struct {
	// Parameters.
	a int
	b int
	// Local variables.
	result int

	resultAddress *int
	returnContext func()
}
```

We have to convert the function bodies now. Because now we can't just execute some stuff, call a function, and continue where we left, **we need to split each function's body each time we would've used a function call**. Each of those blocks of code will be a separate method on the function's stack frame.

`main` would be split in two: the code before the call to `add`, and the code after. `add` doesn't call any function, so it needs only one method.

```go
func (s *mainStackFrame) start() { ... }
func (s *mainStackFrame) afterCallingAdd() { ... }

func (s *addStackFrame) start() { ... }
```

To convert function bodies, we do this:

* To convert the use of a function parameter or local variable, replace it by its corresponding struct field.
* For a return statement, like `return X`, convert it to:
  
  ```go
  *s.resultAddress = X
  nextContext = s.returnContext
  ```

  `nextContext` is a global variable of type `func()` that our VM will use when returning. Setting it is akin to set PC and SB.

* Finally, for a function call, like `r := f(a, b, c)`, convert it to:

  ```go
  nextFrame := &fStackFrame{
  	arg1: s.a,
  	arg2: s.b,
  	arg3: s.c,
  	resultAddress: &s.r,
  	returnContext: s.afterCallingF,
  }
  nextContext = nextFrame.start
  ```

  Here we cheat a little and use a local variable, `nextFrame`. This will temporarily hold a reference to the new stack frame, ie. the address we're going to set our virtual SB to. We pass the arguments and the result address, and we set the new stack frame's return context to `s`, the current stack frame, as SB, and to `afterCallingF` as return address.

OK, let's convert `main` and `add` then:

```go
func (s *mainStackFrame) start() {
	// a := 3
	s.a = 3
	// b := 4
	s.b = 4

	// c := add(a, b)
	nextFrame := &addStackFrame{
		a:             s.a,
		b:             s.b,
		resultAddress: &s.c,
		returnContext: s.afterCallingAdd,
	}
	nextContext = nextFrame.start
}

func (s *mainStackFrame) afterCallingAdd() {
	// return c
	*s.resultAddress = s.c
	nextContext = s.returnContext
}

func (s *addStackFrame) start() {
	// result := a + b
	s.result = s.a + s.b
	// return result
	*s.resultAddress = s.result
	nextContext = s.returnContext
}
```

Now, we need to actually run this from our VM's `main` function. `main`s mission is extremely simple: just set `nextContext` to our simulated `main`, and call `nextContext` repeatedly while it is set, as `main` is the only function without a `returnContext`, and returning from it sets it to `nil`.

```go
var nextContext func()

func main() {
	var exitCode int
	nextFrame := &mainStackFrame{
		resultAddress: &exitCode,
	}
	nextContext = nextFrame.start
	for nextContext != nil {
		nextContext()
	}
	fmt.Println("program exit code:", exitCode)
}
```

And that's it, [see it running!](http://play.golang.org/p/addfnS6jnS)

![Diagram of how all this operates in the real stack.](http://k16.imgup.net/wwwGIFCrea0098.gif)

This is an animation of what's actually going on. Note how similar it is to the animation [from the previous section](#what-is-the-call-stack), except now call stacks live on the heap.

### Conclusion

Our code needs a tiny bit of glue to bridge our explicit call stack with the implicit call stack the language imposes on us, but, apart from that, **the generated machine code** for our simulation and for a real program **is pretty much the same**, save for a few details like allocation (our stack frames are allocated in the heap) and a more indirect handling of SB and PC.

I hope this little exercise illustrates how the call stack works, and how code, in the end, is just data and instructions. I thought it was funny this raw fact leaks into highly abstracted imperative languages as Swift and Rust.

Check [a Fibonacci example](http://play.golang.org/p/RvSGfehpmp) if you want more.
