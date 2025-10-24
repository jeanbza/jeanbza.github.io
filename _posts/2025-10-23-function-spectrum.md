---
layout: post
title:  "Inlining function code"
date:   2025-10-06 02:55:23 -0600
categories: 
toc: true
---

_Preface: John Carmack's
[email on inlining code](https://cbarrete.com/carmack.html) on this subject was
a really impactful read for me. I highly recommend it._

I've lately been working with a new college grad on my team, and he asked a good
question about how often we should be abstracting code into functions. I thought
about it a while and came up with this answer. I'm somewhat satisfied by it, but
if you have a better way to frame this, please reach out to let me know.

## Cost

All abstractions come with a **cost**:

- They introduce complexity.
- They demand context switching.
    - For functions: This cost can be very high when trying to understand a
    high-level function that requires context switching between many nested
    sub-, sub-sub-, sub-sub-sub-, [...] -functions.
- They increase the size of the mental map required to understand the code.
- They may make debugging harder (mostly as a result of all the above).
- They may impact build and runtime performance.
- See also: [there are no zero-cost abstractions](https://www.youtube.com/watch?v=rHIkrotSwcc).

So, their cost should be justified by their value.

## Objective value

Functions are **objectively valuable** in a few situations:

- When they're used to deduplicate logic (my personal bar is logic found in
3 places, but some people go for 2).
- When they're used to extract some logic that needs to be called from
elsewhere (ex a library; although this often falls into the prior bullet).
- When they're used to extract some logic that you want to individually
test.
- (There are probably a few other objectively useful cases)

## Subjective value

Functions are **subjectively valuable** when they're used just for the sake of
abstraction. That subjectivity is what this article is about. Let's dive into
that subjectivity.

Let's imagine a 1000 line program whose logic has no objective reasons to
introduce functions: there's no duplication, there's no logic that needs to be
called from elsewhere, there are no tests, etc.

One type of person might prefer 0 functions: a single main func that is 1000
lines long, each line a valuable statement of business logic. Every thing the
program does can immediately be grokked in perfect context. Debugging is
extremely easy: you can go to the faulting line and perfectly work backwards to
see how you arrived at the fault. And, it's hard to misuse sub-functions: there are none!

Another type of person might prefer so many functions that every business logic
line gets its own function, the caller of which gets its own function, the
caller of which gets its own function, [...], and main is just 2-5 function
calls long. (This used to be a really popular thing in Javascript and Ruby, but
I've noticed
[a lot of pushback on that kind of thing recently](https://rubystyle.guide/#no-single-line-methods))
The advantage here is that it's really easy to get a quick, vague sense of what
main does: A, B, C. Ok but what _actually_ does A do? Oh, it does x, y, z. Ok...
but what does x _actually_ do? Oh, it does i, ii, iii, and iv. Ok..... what does
i _actually_ do? [...] Oh, it sends a GET request to /foo.

So those are two extremes. In between the extremes is a spectrum.

I think the people who tend to want to understand a program _completely_ are
more on the top part of the spectrum (less abstractions to dig through when
trying to understand _completely_) and people who tend to want to understand a
program _quickly_ are more on the bottom part of the spectrum (more abstractions
to gain vague understanding quickly).

## Illustrating subjectivity

To illustrate that a little more:

Person #1 wants to understand program #2 _completely_. Because of all the
abstractions, he has to do a depth first search and collect all leaf nodes (the
valuable business logic) into a mental list before he's able to follow a path
from main start to main end. (And, every time he returns, he has to perform some
of that DFS again as the mapping inevitably falls out of his mind each time he
context switches)

Person #2 wants to understand program #1 _quickly_. Because of the lack of
abstractions, she has to try to map sections of code into logical groupings in
her mind, and can only reason about the order of those groupings mentally. (She
may make a series of notes or flow charts outside of code to help cope) (And,
every time she returns, she has to perform some re-grouping as the mapping
inevitably falls out of her head each time she context switches)
