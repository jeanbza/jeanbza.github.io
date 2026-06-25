---
layout: post
title:  "Designing CRUD operations in Go"
date:   2019-06-08 15:55:23 -0600
categories: 
toc: true
---

This post is intended to provide some insight into the considerations that
go into adding CRUD operations in a Go client library.

## The Problem: Ambiguity

Every time we add a new create or update RPC - `CreateFoo` or `UpdateFoo` - that
takes parameters, we have to ask the questions:

- Is it possible to perform a delete a parameter when updating Foo?
- How should users ignore that parameter when updating Foo?
- How should users delete that parameter when updating Foo?

Consider the following RPC:

```go
message FooUpdateRequest {
  google.protobuf.Duration ttl = 1;        // When set to 0, deletes TTL.
  google.protobuf.Duration expiration = 2; // When set to 0, deletes expiration.
}

service Foo {
  rpc UpdateFoo(FooUpdateRequest) returns (SomeResponse)
}
```

Building a manual layer wrapper around this might look very similar:

```go
type FooConfigToUpdate struct {
  Ttl        time.Duration
  Expiration time.Duration
}

func (f *Foo) Update(ctx context.Context, cfg *FooConfigToUpdate) error {
  // ...
}
```

This seems fairly innocuous, but consider those questions again:

- Is it possible to perform a delete a parameter when updating Foo?
  A: Yes, both parameters in the .proto definition mention that 0 is used as a delete.
- How should users ignore that parameter when updating Foo?
  A: Set it to time.Duration(0).
- How should users delete that parameter when updating Foo?
  A: Uhh... also set it to time.Duration(0)?

As you can see, there's currently no way to distinguish between Delete and
Ignore. That is, if a user passes:

```go
f.Update(ctx, &foopkg.FooConfigToUpdate{Expiration: 5 * time.Second})
```

It's clear that we need to update Expiration to 5s, but what do we do with ttl?
Its default value is `time.Duration(0)` - do we delete it, or ignore it?

We need a way to get around this. Broadly, there are three options we use in client libraries:

- Sentinel values.
- Pointers.
- Optionals.

## Sentinel Values

Sentinel values are basically special values that signal to the client library
to perform special logic. For example, consider:

```go
var NeverExpire time.Duration = -1 * time.Second
```

No user would ever specify -1s as an expiration value, so the library picks that
value as its sentinel. Then, any time the library sees -1s, it knows this is the
special value used to indicate "delete".

If the empty value is passed, the library ignores the operation. If the user
passes the sentinel value, the library performs the delete. A user uses the
sentinel to delete as such:

```go
f.Update(ctx, &foopkg.FooConfigToUpdate{Expiration: foopkg.NeverExpire})
```

## Pointers

When using pointers, we automatically resolve ambiguity, because beside the
empty value we now also have the nil value.

The nil value is taken as "ignore this" (unspecified), and the empty value is
taken as "delete this".

A user uses the empty value to delete as such:

```go
f.Update(ctx, &foopkg.FooConfigToUpdate{Expiration: new(time.Duration(0))})
```

## Optionals

Optionals are a Java concept. Without involving pointers, they add an
additional "set or unset" parameter to a value in a similar way that pointers
are set or nil.

Unlike Java, however, optionals are not part of the stdlib. In
`cloud.google.com/go`, we've created a very light wrapper around `interface{}`
to implement optional types. Since they're light wrappers around `interface{}`,
values are nil-able.

Like pointers, the nil value is taken as "ignore this" (unspecified), and the
empty value is taken as "delete this".

A user uses the empty value to delete as such:

```go
f.Update(ctx, &foopkg.FooConfigToUpdate{Expiration: time.Duration(0)})
```

## Choosing The Right Solution

There are downsides to each solution:

- Sentinels add to the API surface, and reduce the range of values users can
use. They are not described by the type system, like pointers: users have to
read docs to find out how to use them. Finally, they may surprise users using
the reserved value without knowing that they are.
- Pointers only work on types - ints, strings, bools, and other primitives
become quite burdensome to use. Furthermore, pointers can't be added in a
backwards-compatible fashion, since they change the signature of a
parameter/method.
- Optionals are untyped (since they're wrappers around interface{}), and not
idiomatic Go.

On the other hand, the advantages to each solution:
- Sentinels can be added in a backwards-compatible fashion, since they don't
change the signature of a parameter/method.
- Pointers are very idiomatic to Go, widely used, and their semantics are
encoded in the type system, making them very easy and obvious to use.
- Optionals can be added (but not removed) in a backwards-compatible fashion,
since their untyped nature is compatible with an pre-existing inputs supplied
by the user. 

Sentinels should generally be avoided, since we try to minimize API surface
additions. They are commonly used in "uh oh" moments where we have to go amend
a stable client's API surface in which the to-be-fixed portion is already
released.

Pointers are generally the preferred choice but fall short on primitive types,
and aren't backwards-compatible. Optionals are fairly widely used, too.

In general, it's best to investigate the client you're working on and focus on
consistency of choice. Ideally, all clients would be consistent with each other, too.

## A Note On // Experimental

Note that if a part of the API surface is marked experimental, the above notes
on backwards compatibility don't apply. Therefore, if you're adding to an API
surface and not sure how it'll evolve over time, opt to add the
`// Experimental` tag:

```go
// It is EXPERIMENTAL and subject to change or removal without notice.
```