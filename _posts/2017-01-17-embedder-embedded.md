---
title: An idea for accessing embedder from embedded in Go
layout: post
---

Go doesn't have inheritance (subclassing) in the OOP sense. Inheritance has multiple (too many?) uses, but if you want something along the lines of "have a type be like other type, but overriding some parts of it or extending it", then what comes closest is [embedding](https://golang.org/doc/effective_go.html#embedding).

However, there are a few differences between embedding and subclassing. One of them, perhaps the most significant, is this:

> There's an important way in which embedding differs from subclassing. When we embed a type, the methods of that type become methods of the outer type, but when they are invoked the receiver of the method is the inner type, not the outer one.

So if we have a type `Foo`, and a type `FooWrapper` that embeds `Foo`, and both have a method `String`, and `Foo` has a method `Bar` that calls `String` on its receiver, then it will always call `Foo.String`, never `FooWrapper.String`, even if you call it like this: `var v FooWrapper; v.Bar()`.

The opposite behavior, ie. that accessing a field on a method receiver gets you the "overriden" version if it exists, is called [open recursion](http://journal.stuffwithstuff.com/2013/08/26/what-is-open-recursion/). Arguably, this feels more "magical" and prone to spooky action at a distance than Go's straightforward behavior (`this` is always exactly the concrete type you expect where you expect it), and, in its typical OOP incarnation, forces a subtype relationship between the two types, which easily gets awkward.

So I was wondering, what would it take to bring this into Go? Can we do what we really want, ie. accessing the embedder's fields from the embedded type's methods, while retaining the explicitness that Go so values?

## A second method receiver

I've come up with this approach: **let methods have two receivers**.

* The **first receiver** works exactly as Go's current method receiver.
* The **second receiver** refers to **the value that embeds, or directly is, the first method receiver**.

A value will have a method with two receivers in its method set only when these two rules are true:

* If its first receiver were the only receiver, the method would be too in the value's method set (following current Go's rules).
* One of the following rules about the second receiver is true:
  - The value is [assignable to](https://golang.org/ref/spec#Assignability) the type of the second receiver.
  - If it were the only receiver, the method would be too in the value's method set (following current Go's rules).

Let's see how this works out in code, building up from the example exposed above:

{% highlight go %}
type Foo struct {}

func (f Foo) String() string {
	return "I'm a Foo!"
}

func (f Foo) Bar() {
	fmt.Println("Here's my string: " + f.String())
}

type FooWrapper struct {
	Foo
}

func (fw FooWrapper) String() string {
	return "I'm a FooWrapper!"
}
{% endhighlight %}

So from that, if we do this:

{% highlight go %}
var v FooWrapper
v.Bar()
{% endhighlight %}

We see:

```
Here's my string: I'm a Foo!
```

Let's add now a method with two receivers.

{% highlight go %}
func (f Foo, s fmt.Stringer) BarWithTwoReceivers() {
	fmt.Println("Here's my string: " + f.String() + ", and the one of the variable I was called on: " + s.String())
}
{% endhighlight %}

So if we do this:

{% highlight go %}
var v FooWrapper
v.BarWithTwoReceivers()
{% endhighlight %}

We see:

```
Here's my string: I'm a Foo!, and the one of the variable I was called on: I'm a FooWrapper!
```

What happened there is that `s` was assigned to `v`, because `FooWrapper` is assignable to [the `fmt.Stringer` type](https://godoc.org/fmt#Stringer).

## Everything still fits together

### Methods as functions

Just as we can use a method as a function by passing its receiver as first argument, we naturally can call a two-receivers method by passing the second one as second argument:

{% highlight go %}
var v FooWrapper
FooWrapper.BarWithTwoReceivers(v.Foo, v)
{% endhighlight %}

### Satisfying interfaces

A two-receivers method is no special case about when a value satisfies an interface. For example, we can implement `Foo.String` as a two-receivers method and it still would satifsy 

{% highlight go %}
func (f Foo, v fmt.Stringer) String() string {
	return "I'm a Foo! Here's the string of what I was called on: " + v.String()
}

var v FooWrapper
var stringer fmt.Stringer = v
{% endhighlight %}

### Second receiver when not embedding?

Note that, becuase `Foo` itself is a `fmt.Stringer`, a `Foo` also has the `BarWithTwoReceivers` method! The second receiver plays nice with (and, really, is only useful when using) embedding, but doesn't require it. In this case, both receivers would refer to the same underlying variable.

### Non-inteface second receiver

Although using an interface as second receiver type is the most natural choice on most occasions, and the most flexible, there's no reason you can't use a concrete type:

{%highlight go%}
func (f Foo, fw FooWrapper) OnlyForFooWrapper() {}
{% endhighlight %}

Although, admittedly, that's really just this:

{%highlight go%}
func (fw FooWrapper) OnlyForFooWrapper() {
	f := fw.Foo
}
{% endhighlight %}

The key about open recursion is that it's _open_, for what it needs some kind of runtime dispatch. In C++, that means virtual methods, and in Go's case we use interfaces. Maybe we can restrict second receiver types to interfaces, but I feel that restriction would hurt orthogonality a bit.

## Conclusion

So what do we achieve with this?

* Well, open recursion! The ability to access the "actual" receiver from an embedded method.
* Not all methods feature open recursion. You opt in by specifying a second receiver, retaining control over your API's extensibility and over dispatch strategy (static or dynamic). It roughly maps to declaring a method as virtual in C++ or _not_ declaring it as final in Java, I think.
* You still have access to the non-virtual receiver! I know of no other language that allows this.
* It's a strict extension of Go. Every current Go program would be valid and work the same.
* No inheritance needed! Inheritance couples together method dispatch and type relationships. We keep both straightforward by leveraging Go's already existing way of doing virtual dispatch via kind-of-subtyping: interfaces.

What about drawbacks? Honestly, my biggest issue with this is that I struggle to come up with or find use cases significant enough to warrant such a change to the language. But, anyway, if we hypothetically wanted to do this, here's a possible approach!
