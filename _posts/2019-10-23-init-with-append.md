---
layout: post
title:  "Bad Times Initializing With Append"
date:   2019-10-23 15:55:23 -0600
categories: [go]
description: "Initializing a variable can be a bad idea."
toc: true
---

I recently found out as part of [a code review](https://go-review.googlesource.com/c/tools/+/184357)
that initializing variables with append is, under some circumstances, A
Terrible Idea. This post briefly explores what can happen when you do that,
the underlying reason these things are happening, and how to avoid it.

# A modest test

Consider [the following program](https://play.golang.org/p/3v_fgJLmtgO):

```
package main

import (
	"fmt"
)

func main() {
	a := []string{"one", "two", "three", "four", "five"}
	b := append(a, "six")
	c := append(a, "seven")

	fmt.Println("a", a)
	fmt.Println("b", b)
	fmt.Println("c", c)
}

```

What do you think the value of a, b, and c should be?

```
a [one two three four five]
b [one two three four five six]
c [one two three four five seven]
```

That make sense! Now consider [this very similar program](https://play.golang.org/p/6EShDJVSgfo):

```
package main

import (
	"fmt"
)

func main() {
	a := append([]string{"one", "two", "three", "four"}, "five")
	b := append(a, "six")
	c := append(a, "seven")

	fmt.Println("a", a)
	fmt.Println("b", b)
	fmt.Println("c", c)
}
```

Do you think the values are any different? If you guessed or assumed there's no
difference - as I did in my codereview - you would be mistaken. The values are,

```
a [one two three four five]
b [one two three four five seven]
c [one two three four five seven]
```

# What the heck?

Note: this section assumes you understand that slices are composed of a
reference to an underlying array, a length, and a capacity. See more details at
https://blog.golang.org/go-slices-usage-and-internals.

The difference is that the first array was initialized by `[]string{...}`, and
the second by `append(...)`. Why does that make a difference, though, and what's
going on under the hood?

Well, if we inspect the length and capacity of `a` with
`fmt.Println("len:", len(a), "cap:", cap(a))` in both cases, we'll see that
under the hood there's another small difference:

```
# program 1 - []string{...}
len: 5 cap: 5
# program 2 - append(...)
len: 5 cap: 8
```

When we use `[]string{...}`, the resultant slice's length and capacity are
equal. However, when we perform `append([]string{...}, "something")`, we are
guaranteed to get a slice with spare capacity, since the append must immediately
grow the array (the inner slice has exact length=capacity - no space remaining).

So, in the first program, both times that we appended the length already equaled
the capacity, and a new underlying array had to be created for each append. That
meant that both appends added a value to separate underlying arrays.

However, in the second program, there was room in the underlying array - length
was less than capacity. So, instead of creating new arrays for both appends,
both appends operated on the same underlying array. When we assigned
`b := append(a, ...)`, `b` got its own `len: 6, cap: 8` counters but used the
array underlying `a`. Similarly, when we assigned `c := append(a, ...)`, `c`
got its own `len: 6, cap: 8` and used the array underlying `a` also. Since `b`
and `c` share an underlying array, but _do not_ share their counters, that means
they both tried to assign a value to index `5`.

# How to solve this?

Here are some other ways you could accomplish the same thing:

- If you know the values ahead of time, do `a := []string{...}`.
- If you know the capacity ahead of time, do `a := make([]string, 0, cap)`
- Copy https://github.com/golang/go/wiki/SliceTricks#copy.

# When is this not a problem?

The key piece above is that `b` and `c` are the result of appending to `a`. If
`a` is only ever going to be appended to itself (a very common use of slices),
the problem won't occur, since there will always only ever be one length counter
to maintain. With only one length counter, there's no room for two appends
accidentally assigning to the same index.