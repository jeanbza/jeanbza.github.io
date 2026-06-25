---
layout: post
title:  "testing in Go"
date:   2022-02-08 15:55:23 -0600
categories:
toc: true
---

This is part of a series of posts on testing in Go:

- [Stubbing gRPC clients in Go tests](/2020/10/08/stubbing-grpc-clients)
- **testing in Go**
- [Complex assertions](/2022/02/08/complex-assertions)

Go has a ["testing" library](https://pkg.go.dev/testing) in the stdlib, and [is opinionated about how to use that library to write tests](https://go.dev/doc/tutorial/add-a-test). Despite that, a fair amount of projects choose to use other libraries for their testing and test assertions.

This post was inspired by a long internal conversation within the Google Go readability crowd recently, in which [we collectively re-affirmed our position to disallow](https://google.github.io/styleguide/go/decisions#assert) assertion libraries other than testing in Go at Google.

I'm not going to rehash that conversation in full here. But, I wanted to briefly touch on why I think that people _do_ reach for non-"testing" libraries, and why I think that you should instead generally reach for using "testing".

For context, I've used several non-"testing" libraries on pre-established projects, including [testify](https://github.com/stretchr/testify), [ginko/gomega](https://github.com/onsi/ginkgo), and [check](https://github.com/go-check/check). I've also used "testing" on many projects, both internal to Google, and on the Go project, and in numerous open source projects. So, I think that I've seen a fair amount of both sides.

## Why do people use non-testing libraries

In my experience, people tend to use non-"testing" libraries for one or some of these three reasons:

- LibraryX is more terse than using Go conditionals and "testing".
- LibraryX feels more BDD than Go conditionals and "testing".
- LibraryX feels like Lang-Y (often Java/JUnit).

Let's evaluate and talk about each of these. I'll use testify to compare against "testing", but you can substitute most of what's below with any LibraryX of your choice.

## Terse

Here is a simple "testing" test:

```go
func TestFoo(t *testing.T) {
    if got, want := Foo(), 5; got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}
```

Here is the same test with testify:

```go
func TestFoo(t *testing.T) {
    assert.Equal(t, 5, Foo()) // We saved 2 LoC.
}
```

The argument for terseness is very valid and similarly often repeated about error checking ([ex1](https://github.com/golang/go/issues/32437), [ex2](https://github.com/golang/go/issues/21161), [ex3](https://github.com/golang/go/issues/71203)).

## BDD

Some folks strongly prefer writing tests with assertions that read more like ordinary human sentences. That style is often associated with [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development) or [Gherkin-style testing](https://en.wikipedia.org/wiki/Cucumber_(software)).

I actually used to work at a consultancy (Pivotal) that was fairly big on pushing this. The idea was that PMs and other non-Programmer types would be able to read and perhaps even write tests, and that the tests would be this external contract to the code.

I've never once seen this play out as described in practice, so I have to admit to being a skeptic. But even then, some programmers are used to it and just prefer the style over the more "ordinary code" style the ordinary "testing" tests use, with explicit conditionals, loops, and general programming happening in the test.

## LangX port

Some folks coming from LangX are comfortable with the testing idioms in that language, and like to port them to Go. AFAICT that's probably how [testify started in 2012](https://github.com/stretchr/testify/commit/2930d903bf969b4e153bb2f2d05a65455d214949) as basically a port of JUnit4:

```
assertEquals(expected, actual) // JUnit4
assert.Equal(t, expected, actual) // testify
```

## Functionality gaps, ecosystem drag, and expressiveness

I've not seen other reasons to use non-"testing" libraries to test Go code. I'm not aware of any functionality that they provide that you don't have available to you with ordinary Go code and "testing".

And in fact, I'd say that they provide _far less_ functionality and expressiveness. I call this ecosystem drag: libraries struggle to maintain feature parity with Go advancements.

These libraries often don't support things like [parallel testing](https://pkg.go.dev/testing#T.Parallel), [subtests](https://pkg.go.dev/testing#hdr-Subtests_and_Sub_benchmarks), and [benchmarking](https://pkg.go.dev/testing#hdr-Benchmarks); they struggle to keep up with Go advancements like [fuzz testing](https://pkg.go.dev/testing#hdr-Fuzzing), [synctest](https://go.dev/blog/synctest), and generics. As Go continues to advance and develop new features, libraries tend not to keep up.

And in terms of expressiveness, libraries by definition can't express the full range of things that ordinary Go code can. Worse, in trying to do so by flattening the Go language into assertions (ex [testify has 312 assertions](https://pkg.go.dev/github.com/stretchr/testify/assert) as of writing), they can end up with a considerably more complex and less easy to use alternative than just writing Go code.

## Prefer testing

I sympathise with all three benefits to using non-"testing" libraries. But, I tend not to agree with the motivations:

- Porting idioms to Go is pretty self-explanatory: Go is not LangX.

- Terseness-[by-way-of-magic](https://github.com/stretchr/testify/blob/2a57335dc9cd6833daa820bc94d9b40c26a7917d/assert/assertions.go#L164) runs opposite to how Go is designed to be written. [Go's design philosophy is expressly to be simple](https://go.dev/doc/effective_go#introduction) and non-magical, sometimes at the cost of some boilerplate. I'm not mentioning this to be a language purist: I mention this because if you're fighting any given language's design ethos, you'll often find yourself writing code that is harder for your peers to understand, and sometimes more code than you needed, or worse code than what you'd have ended up with using that language's idioms.

- And BDD [is not a Go idiom](https://go.dev/doc/tutorial/add-a-test). I do however understand this motivation if it's a _cultural_ idiom for someone, or their team, or their company.

My recommendation is to lean into Go's design ethos and use "testing" and ordinary Go in your tests. I think that the advantages far outweigh what you lose above:

- You keep the full, huge range of "testing" functionality and Go expressiveness.
- Your tests can be read by newcomers to your project, since they're ordinary Go code. No library knowledge required.
- Your tests depend only on the stdlib, and are all but guaranteed to be fast and [free of bugs](https://github.com/stretchr/testify/issues?q=is%3Aissue%20state%3Aopen%20label%3Abug) (or, if not, it'll be fixed very soon!).

These benefits are massive, and in my opinion far outweigh what you'd get otherwise.
