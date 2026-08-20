---
layout: post
title:  "Google styleguide code review skills"
date:   2026-08-13 02:55:23 -0600
categories: [ai, claude]
description: "A plugin containing skills for Claude to perform Google's styleguide reviews."
toc: true
---

At Google, code changes require both a language readability and a domain review.
The former is what you'd expect from a code review, whereas the latter was a
review from someone[^1] very proficient in the language, intended to help drive best
practices of that language across the company, and sometimes to drive
conformance towards a certain style of the language.

Each language maintained a style guide: a list of best practice and style
decisions, curated by the hundreds or thousands of readability experts for that
language.

I was a C++ and Go readability reviewer, and contributed some tidbits to the Go
styleguide, as well as to many discussions about how to write Go. The styleguide
was a great place to put best practices for the language, but also a great place
to go to learn, since best practices were usually followed with an explanation.
That way, newcomers to the language could ask for a change and simultaneously
reference the styleguide as the high-quality explanation for the change request.

Google has _very_ generously open sourced the styleguides for its languages at
[https://google.github.io/styleguide/](https://google.github.io/styleguide/). I
highly encourage practitioners of any of the languages there to give them a
read. I can greatly recommend the C++ and Go ones, but I'm sure the others are
good too.

For the last 6 months at Netflix, when asking Claude to write code, I've found
it varying degrees of helpful to have Claude read those styleguides, and apply
edits based on advice in them. It's not been perfect - far from it,
unfortunately. But it's generally been better than not having it do so.

So, I thought I'd open source it:
[https://github.com/jeanbza/claude-extensions](https://github.com/jeanbza/claude-extensions).

To install it, simply run the following inside a Claude session:

```
/plugin marketplace add jeanbza/claude-extensions
/plugin install google-go-review@jeanbza
/reload-plugins
```

To use it, simply run `/jeanbza:google-go-review` (or cpp, or py - the other
two I've added so far).

There's more for me to do, in terms of prompt engineering: Claude frequently
does a poor job at ["Handle errors"](https://google.github.io/styleguide/go/decisions#handle-errors)
for example. (My pet theory is that Claude doesn't bother to check the
signatures) I'll keep working on it, both for myself and whomever else wants to
use it. I'm also working on a project to go a bit lower down the stack and try
to inject some evaluators into its reward model with lifecycle hooks. But for
now, I hope it's helpful to you as it's been for me!
