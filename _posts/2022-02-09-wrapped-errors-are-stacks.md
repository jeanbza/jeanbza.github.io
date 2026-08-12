---
layout: post
title:  "Wrapped errors are stacks"
date:   2022-02-09 15:55:23 -0600
categories: [errors, go]
description: "Wrapped errors form a stack."
toc: true
---

# Wrapped errors are stacks

Wrapping an error creates a stack: a linked list of error pointing to the next error, where newly wrapped errors are added to the head, and the stack is traversed from head to tail during As, Is, and with the Unwrap interface.

It's important to keep this in mind when you're designing the internal representation for your unwrappable structured error. The simplest representation is a single error:

```
type decompressErr struct {
  name string
  err error // Points to the next error down the stack.
}

func (e *decompressErr) Error() string {
  return fmt.Sprintf("decompress %s: %s", e.name, e.err)  
}

func (e *decompressErr) Unwrap() error { return e.err }
```

Most unwrappable structured errors should only contain a single error. They have obvious semantics and are easy to use.

Unwrappable structured errors that contain multiple errors have much less clear semantics. Consider:

```
// We recommend against this approach.
type PathParseErrors struct {
  // A map of path to parse error for that path.
  errors map[string]error
}

func (e *PathParseErrors) Error() string {
  return fmt.Sprintf("%v", e.errors)
}

func (e *PathParseErrors) Unwrap() error {
  // Nothing we return here will be obvious.
}
```

In this example, the stack semantics are broken: we have an unwrappable structured error that contains a map of errors. But, which error will Unwrap return? There's no right answer: any choice would be non-obvious to the user. A slice has the same issue as a map: there's no obvious error to return.

This issue is exacerbated by the fact that it's impossible for the author of PathParseErrors to document their Unwrap method in a way that will directly help users. Users often don't interact with the Unwrap method directly: they use tools like As and Is, which themselves call the Unwrap method. And, this error may exist in a library that is several layers deep in a dependency tree: a user may have a very hard time finding the exact library whose documentation to go read when they've got an opaque stack of wrapped errors.

When you need to collect several errors at once, use `[]error`, a map of errors, or a structured error that does not support Unwrap.