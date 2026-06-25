---
layout: post
title:  "Error unwrapping for library authors"
date:   2022-02-08 15:55:23 -0600
categories: 
toc: true
---

# Introduction

In https://go.dev/blog/go1.13-errors, Go introduced new language changes that improved how errors are handled in Go programs. These changes include the concept of wrapping errors, as well as tools and semantics for unwrapping errors. They have enabled a rich ecosystem of detailed errors, and given users more ways to examine errors..

Having had a few years to learn about how these tools can be used, let's discuss some of the subtleties of wrapping and unwrapping, with an eye towards compatibility of our APIs. Accordingly, this article is particularly relevant to library authors, as we will often view wrapped errors through the lens of API compatibility.

# Wrapped errors are part of your public API

In "Working with Errors in Go 1.13", the authors wrote that "When adding additional context to an error, either with fmt.Errorf or by implementing a custom type, you need to decide whether the new error should wrap the original.". There are numerous considerations that factor into whether to wrap an error, but library authors should specifically consider that wrapped errors become part of their public API.

Let's re-examine one of the examples from that article:

```
func Decompress(name, path string) error {
  // ...
  if err != nil {
    return fmt.Errorf("decompress %v: %v", name, err)
  }
  // ...
```

When we write error returns like this in our public APIs, users are given textual information but no structured information. They have no way of inspecting more than the string contents of the error handed to them: as the article states, "Creating a new error with fmt.Errorf discards everything from the original error except the text.". We are therefore able to change any part of the error returned without causing any backwards incompatible changes, assuming users aren't relying on the string contents of these errors.

Let's now alter this example to return a structured, unwrappable error:

```
type PathParseError struct {
  Path string
}
func (e *PathParseError) Error() string {
  return e.Path + ": could not be parsed"
}

// ...

func Decompress(name, path string) error {
  err := &PathParseError{Path: path}
  if err != nil {
    return fmt.Errorf("decompress %v: %w", name, err)
  }
  // ...
}
```

`We've now given` users powerful new capabilities for introspecting the returned error. Users now have structured information that they can rely on at runtime, such as the existence of PathParseError, and its fields; whereas before they could only string match the error contents, a practice that is usually discouraged.

But exposing that information to users comes with responsibility. When we only return unstructured errors, we often can be liberal in changing the string-only contents of our errors. However, when we return structured errors to users, we have to be more careful: removing the %w, or changing its value, will be noticeable behavior changes that affect how user code will interpret the returned error.

The subtlety here is that though the structure of the returned error changes when we remove %w or change its value, the type information remains the same - we still just return an error, not noticing that its underlying information has changed. Therefore users won't observe these changes at compile time: they'll observe them at run time.

To combat this, here are some tips:

- Users should only rely on structured errors from stable libraries that they trust not to change.
- Authors of stable libraries should aim to preserve the behavior of their structured error returns. They should write tests that exercise all observable behavior to aid in that goal, and document which structured errors users can expect to interpret the returned error as. 

# Wrapping external library errors

Let's take this example a bit further by wrapping errors from external libraries in our code:

```
package externaldep

type PathParseError struct {
  Path string
}

func ParsePath(path string) (string, error) {
  // ...
}
```

```
package mycode

func Decompress(name, path string) error {
  _, err := externaldep.ParsePath(path)
  if err != nil {
      return fmt.Errorf("decompress %v: %w", name, err)
  }
  // ...
}
```

As before, we have added to our public API by wrapping `externaldep.PathParseError` and returning it as our error. And as before, users can introspect the error we return and get an `externaldep.PathParseError`. But, `externaldep.PathParseError` lives in an external library - we have no control over it! Our users can now be broken by those external authors in the same ways we described above, without any type information changing.

These behavior changes can cause user breakages. The further that breakage happens from the user code, the harder it will be for the user to understand and debug. Each successive layer of external library error wrapping is an entirely different codebase that the debugger has to understand in the context of all the other codebases. The complexity increases quickly!

To combat this, here are some tips:

- Users should prefer relying on structured errors closer to them than farther from them. A structured error that is 5 levels of libraries' wrapping below them has a far greater chance of a breaking behavior change occurring than a structured error 1 level below them.
- Users should prefer relying on structured errors whose chain never relies on an unstable library.
- Authors of libraries should prefer not to wrap external library errors, unless the use case has well understood value and the external library is stable.

# Unwrap is part of your public API

We've talked about the fact that wrapping errors changes the behavior of your public API. Let's now discuss Unwrap, and how it can subtly become part of your public API.

When you return a structured error with an Unwrap method, users can use As, Is, and the Unwrap interface to introspect it and errors below it:

```
err := Decompress(name, path)
var pathParseErr *PathParseError
if errors.As(err, &pathParseErr) {
    // err, or some error that it wraps, is a path parse problem
}
```

Users may not be aware of new layers being added to the chain of errors:

```
type decompressErr struct {
  name string
  err error
}

func (e *decompressErr) Error() string {
  return fmt.Sprintf("decompress %s: %s", e.name, e.err)  
}

func (e *decompressErr) Unwrap() error { return e.err }

func Decompress(name, path string) error {
  _, err := externaldep.ParsePath(path)
  if err != nil {
      return &decompressErr{name: name, err: err}
  }
  // ...
}
```

Here, we've added an intermediary decompressErr. This might be useful somewhere else in our code that calls Decompress, but since it's un-exported, users have no way to use it for introspection.

But, even though it's un-exported, the Unwrap method is part of our public API. If we remove the Unwrap method, for example, the errors.As example breaks: there is no Unwrap link between the returned error and the externaldep.PathParseError. Similarly, if we change Unwrap to behave differently, it will constitute a behavior change in our library.

To combat this, here is a tip:

- In general, keep Unwrap simple and deterministic. It should just return the underlying error. If you find that your Unwrap logic is non-deterministic, or you need logic in Unwrap, it might be a signal that you shouldn't be unwrapping.

# Conclusion

Error wrapping is a powerful tool that provides upstream users a much richer set of functionality for understanding returned to them by downstream libraries. But, it's important to be aware that supporting wrapped errors comes with responsibility for compatibility, like any other part of your API. And it's important to consider whether your error type makes sense to be unwrapped at all.

For most code, it's best to start simple: use %v instead of %w when you annotate errors, and don't provide Unwrap on your custom error types. Wait until you learn about specific use cases for providing unwrap mechanics, and then carefully consider how to support them. We hope the considerations highlighted in this article help guide that decision.
