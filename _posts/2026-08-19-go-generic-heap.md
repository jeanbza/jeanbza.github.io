---
layout: post
title:  "A generic heap in Go"
date:   2026-08-19 02:55:23 -0600
categories: [go, heap]
description: "An example of the heap package, re-imagined as a generic heap."
toc: true
---

Let's re-imagine the venerable [container/heap](https://pkg.go.dev/container/heap)
package with generics instead of an interface.

### Usage without generic methods

[Here's how a user uses heap today:](https://go.dev/play/p/3-QCvtyZVH1)

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }

func (h *IntHeap) Pop() any {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

func main() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
}
```

And, in case you missed it, `Push`, `Pop`, and so on use `any`. So, you could
write `heap.Push(h, 3)` and then `heap.Push(h, "foo")` and it'd compile! But
you'd get a panic at runtime:

```
$ go build .
# aww
$ go run .
panic: interface conversion: interface {} is string, not int
```

Overall, it's definitely usable and great not to have to re-implement heap
bubbling, but the UX is a bit clunky. It's not great to lose types, and it's not
great to have to implement 5 functions every time I have a different type of
heap.

### Usage with generic methods

With generic methods, a differently-defined heap (which we'll look at shortly)
[could just be:](https://go.dev/play/p/GiRznCugGt-)

```go
package main

import (
	"container/heap"
	"fmt"
)

func main() {
	h := &heap.Heap[int]{2, 1, 5}
	h.Init()
	h.Push(3)

	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", h.Pop())
	}
	fmt.Println()
}
```

So much easier! No more having to define 5 methods per type: we just declare a
heap[<type>] and use it.

And, we're using generics instead of `any`, so we have types on our methods. If
we try to do `h.Push("foo")`, we get a nice compiler error:

```
$ go build .
# example.com/foo
./main.go:20:9: cannot use "foo" (untyped string constant) as int value in argument to h.Push
# yay!
```

### Implementing heap with generic methods

And finally, here's how the heap package looks with generic methods. Mostly the
same as it is, except we define Push and Pop directly on a heap type rather than
as free functions that take an interface. And, cmp aleady gives us `cmp.Ordered`
as the type constraint, so we can just use that directly. The bubbling logic
remains the same:

```go
import "cmp"

type Heap[T cmp.Ordered] []T

func (h Heap[T]) Len() int           { return len(h) }
func (h Heap[T]) Less(i, j int) bool { return h[i] < h[j] }
func (h Heap[T]) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *Heap[T]) Init() {
	n := len(*h)
	for i := n/2 - 1; i >= 0; i-- {
		h.down(i, n)
	}
}

func (h *Heap[T]) Push(x T) {
	*h = append(*h, x)
	h.up(len(*h) - 1)
}

func (h *Heap[T]) Pop() T {
	n := len(*h) - 1
	h.Swap(0, n)
	h.down(0, n)
	x := (*h)[n]
	*h = (*h)[:n]
	return x
}

func (h *Heap[T]) Remove(i int) T {
	n := len(*h) - 1
	if n != i {
		h.Swap(i, n)
		if !h.down(i, n) {
			h.up(i)
		}
	}
	x := (*h)[n]
	*h = (*h)[:n]
	return x
}

func (h *Heap[T]) Fix(i int) {
	if !h.down(i, len(*h)) {
		h.up(i)
	}
}

func (h *Heap[T]) up(j int) {
	for {
		i := (j - 1) / 2
		if i == j || !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		j = i
	}
}

func (h *Heap[T]) down(i0, n int) bool {
	i := i0
	for {
		j1 := 2*i + 1
		if j1 >= n || j1 < 0 {
			break
		}
		j := j1
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2
		}
		if !h.Less(j, i) {
			break
		}
		h.Swap(i, j)
		i = j
	}
	return i > i0
}
```
