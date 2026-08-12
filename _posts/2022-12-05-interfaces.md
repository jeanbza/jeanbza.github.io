---
layout: post
title:  "Interfaces in depth"
date:   2022-12-05 01:55:23 -0600
categories: [go]
description: "A deep dive into Go interfaces and method sets, replacing intuition with the actual rules."
toc: true
---

## Foreword

A conversation with a colleague inspired a deep dive into the refspec, to gain
concrete understanding on some things I had until then only had intuitions
about. I ended up writing nearly an article to that colleague in chat
(sorry colleague...), so I thought I'd take it over the finish line and actually
write the article that my chat message was trying to be.

This is that article.

## Impetus

This article is inspired by the following confusion:

```go
type myInterface interface{ hello() }
var m1 myInterface = implementsMyInterface{}
m1.hello() // works!
var m2 *myInterface = &m1
m2.hello() // does not work
```

[play/p/g56XEBk_OLs](https://go.dev/play/p/g56XEBk_OLs)

The core question is: Why can we use `m1`, not `m2`?

**TLDR**: Go veterans will realise that pointer to interface is an anti-pattern.
It represents kind of a misunderstanding of what's going on: the user almost
certainly wants a pointer to the _struct_. Both concrete structs and pointer to
structs can implement interfaces. That's the intuition I mentioned above. But,
let's dive into this a bit and figure out what's behind this.

## What are interfaces, anyway?

From [Laws of reflection](https://go.dev/blog/laws-of-reflection),

> A variable of interface type stores a pair: the concrete value assigned to the
variable, and that value’s type descriptor.

So, I'll simplify this a bit to the hand-wavy description that an interface type
points to a concrete type. For example, consider an interface type that is
implemented by a struct:

```go
var foo someInterface = someStruct{}
```

Here, `foo` is a variable whose type is `someInterface`. Its interface type
"points" ("holds" / "is assigned" / etc) to `someStruct`.

Let's modify that a bit:

```go
var foo someInterface = &someStruct{}
```

Here, `foo` is a variable whose type is `someInterface` which points to a
pointer which points to `someStruct`.

Ok... still in normal territory. Now let's go back to where this question came
from and consider if `foo` was of type `*someInterface` instead:

```go
var foo someInterface = someStruct{}
var bar someInterface = &someStruct{}

var gaz *someInterface = &foo // bad
var urk *someInterface = &bar // also bad
```

Here, `gaz` and `urk` are pointers to an interface. That's almost certainly user
error. Pointers to structs are useful,

- They allow us to avoid copying when passing structs around.
- They allow us to modify and persist struct state.

But, what does a pointer to an interface type give us? Nothing!

- An interface type is already a type of pointer as it is, so there's no copying
when we pass it around.
- An interface type has no state itself. Its implementing type - a struct - can,
but not the interface itself. So there's no modify/persist state benefit.

## Interfaces behave different than structs

Ok, so we've talked about how interfaces are fundamentally different, and that
variables conceptually should (usually) not use interface pointer types; in
constrast to structs, where pointers to structs are very common and useful.

Now let's look at how struct and interface types behave differently with regards
to pointer referencing and dereferencing. Let's do so by collecting together a
few facts about interfaces from the refspec, to prepare for our conclusion:

### A struct can implement an interface with concrete or pointer method receivers

A struct can implement an interface with either concrete or pointer method
receivers. Per
[ref/spec#Interface_types](https://go.dev/ref/spec#Interface_types), there's no
way to specify concrete or pointer method receiver in an interface. (indeed,
it's moot to the interface: the interface defines, well, the interface, not the
implementation details)

Concretely, both these structs implement the interface:

```go
type myInterface interface{ hello() }

type concreteMethodReceivers struct{}
func (m concreteMethodReceivers) hello() {}

type pointerMethodReceivers struct{}
func (m *pointerMethodReceivers) hello() {}
```

[play/p/sjU9d72ZWzw](https://go.dev/play/p/sjU9d72ZWzw)

### Selectors automatically dereference pointers to structs

Selectors automatically dereference pointers to structs [ref/spec#Method_values](https://go.dev/ref/spec#Method_values):

> As with selectors, a reference to a non-interface method with a value receiver using a pointer will automatically dereference that pointer: pt.Mv is equivalent to (*pt).Mv.

So, for implementing interfaces:

- If you have a struct that implements the interface with
_concrete method receivers_, you can use either concrete struct or pointer
to your struct as type for interface (latter will be de-referenced).
- If you have a struct that implements the interface with
_pointer method receivers_, you have to use pointer to your struct as type
for interface (concrete struct _won't_ be automatically turned to pointer).

Concretely:

```go
var f1 foo = concreteMethodReceiverStruct{}
f1.Hello() // works

var f2 foo = &concreteMethodReceiverStruct{}
f2.Hello() // works

var f3 foo = pointerMethodReceiverStruct{}
f3.Hello() // does not work

var f4 foo = &pointerMethodReceiverStruct{}
f4.Hello() // works
```

[play/p/aWQ8C2-SwZ2](https://go.dev/play/p/aWQ8C2-SwZ2)

### Pointers to interfaces do not automatically dereference

Pointers to interfaces do not automatically dereference, like pointers to
structs do. (they used to in pre-Go1, fwiw; [g/golang-nuts/c/RhIIHM3XC4o](https://groups.google.com/g/golang-nuts/c/RhIIHM3XC4o))

Concretely:

```go
var thing myInterface = myStruct{}
thing.Whatever() // works
var thing2 *myInterface = &thing
thing2.Whatever() // does not work
```

[play/p/9QBQmO4-nZN](https://go.dev/play/p/9QBQmO4-nZN)

## Putting it all together

So, let's talk about the various ways you can declare and use an **interface**.
As mentioned before and shown with
[play/p/aWQ8C2-SwZ2](https://go.dev/play/p/aWQ8C2-SwZ2):

- ✅ can implement interface with concrete type method receiver + concrete type
- ✅ can implement interface with concrete type method receiver + pointer type (_auto de-reference_)
- ❌ can implement interface with pointer type method receiver + concrete type (_no auto reference_)
- ✅ can implement interface with pointer type method receiver + pointer type

Now, let's contrast that with the various ways that you can declare and use a
**struct**. As shown with
[play/p/IfD0MGTLT_n](https://go.dev/play/p/IfD0MGTLT_n) and spelled out in
[ref/spec#Method_values](https://go.dev/ref/spec#Method_values):

- ✅ can call concrete type method receiver with concrete type
- ✅ can call pointer type method receiver with concrete type (_auto reference_)
- ✅ can call concrete type method receiver with pointer type (_auto de-reference_)
- ✅ can call pointer type method receiver with pointer type

So, the rules for structs and interfaces are different, to prevent interface
mis-use.

## Afterword

You can read more about how interfaces are represented here:

- [Go Data Structures: Interfaces](https://research.swtch.com/interfaces) by rsc@
- [Go Interfaces](https://www.airs.com/blog/archives/277) by iant@
- [Laws of Reflection](https://go.dev/blog/laws-of-reflection) by r@
