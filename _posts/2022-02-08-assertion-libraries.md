---
layout: post
title:  "Don't use assertion libraries in Go"
date:   2022-02-08 15:55:23 -0600
categories: 
toc: true
---

Foreword #1: This is a very sensitive topic for many people. 🙂 I find that many people have never considered writing tests _without_ assertion libraries. It might come as a shocking surprise to consider such a world. That shock and surprise might be painful. Steel yourself.

Foreword #2: This is inspired by a long conversation the Google Go readability crowd recently, in which we collectively re-affirmed our position to ban assertion libraries in Google. Most of this post is my own responses, tidied up.

# Just stop

Assertion libraries are libraries that attempt to combine the validation and production of failure messages within a test.

This post is not about mocking libraries, or comparison libraries. It is about assertion libraries. Examples of assertion libraries include [github.com/stretchr/testify](https://pkg.go.dev/github.com/stretchr/testify), [github.com/onsi/ginkgo](https://pkg.go.dev/github.com/onsi/ginkgo), [gopkg.in/check.v1](https://pkg.go.dev/gopkg.in/check.v1), and [github.com/franela/goblin](https://pkg.go.dev/github.com/franela/goblin).

Don't use these things. They provide little value over the stdlib, and tend to make your codebase (far) less readable. Use ["testing"](https://pkg.go.dev/testing) for assertions, and comparison libraries like [github.com/google/go-cmp/cmp](https://pkg.go.dev/github.com/google/go-cmp/cmp) to perform the comparison (but not _the assertion_) for more complex objects that can't be compared with basic operators (`==`, `>`, etc).

Sidenote: The [testing](https://pkg.go.dev/testing) library is also an assertion library, but for the sake of this article we'll just call it "the stdlib" or "testing", and refer to all the non-stdlib assertion libraries as "assertion libraries".

# Assertion libraries bring incomprehensibility

Consider the following Java code:

```
assertParagraphElement(bodyStructuralElements.get(25).getParagraph().getElements().get(0), 311, 312, Person.class);
```

What the heck is going on here? I have no freaking idea what this line is asserting. This is _real code_ pulled from a _real codebase_ owned by a well respected team. It's incomprehensible.

Let me show you another one, in C++ this time:

```
 EXPECT_CALL(
        *mock_quota_server_client_,
        MultiGetTokens(
            ::testing::AllOf(
                ::testing::AllOfArray(
                    experiment_ids |
                    ::websitetools::feeds::range::transformed(
                        [quota_group_suffix](ExperimentId experiment_id) {
                          return dos_quotas::HasRequest(
                              absl::StrFormat(kQuotaExperimentGroupId,
                                              quota_group_suffix),
                              absl::StrFormat("%s:%d",
                                              kQuotaUserId,
                                              experiment_id));
                        })),
                dos_quotas::HasRequest(
                    absl::StrFormat(kQuotaUserGroupId, quota_group_suffix),
                    kQuotaUserId)),
            _, _))
        .Times(times);
```

Again, real code. Respectable team. Completely incomprehensible.

(editorial note: I'm using Java and C++ code because when I wrote my emails, I wanted to compare sister languages in Google - which do allow assertion libraries - with Go in Google, which does not. Confusing examples like these are fairly easy to search for on GitHub)

The thing with assertion libraries is that it's never just `assert.Equal(a, b)`, like the advertisement purports. There is a continual gravity towards complexity. There are [_277 assertions in the testify library, as of this writing_](https://pkg.go.dev/github.com/stretchr/testify/assert). 277!!!! That's before you even get into [making your own assertions](https://pkg.go.dev/github.com/stretchr/testify/assert). By the way, the likelihood of your developers - and you! - writing custom assertions approaches 100% as your codebase grows over time. In my experience it approaches 100% awfully quickly; we're talking a few weeks.

# Assertion libraries don't provide value

Let's take a look at the examples on the testify front page:

```
// assert equality
assert.Equal(t, 123, 123, "they should be equal")

// assert inequality
assert.NotEqual(t, 123, 456, "they should not be equal")

// assert for nil (good for errors)
assert.Nil(t, object)

// assert for not nil (good when you expect something)
if assert.NotNil(t, object) {
    // now we know that object isn't nil, we are safe to make
    // further assertions without causing any errors
    assert.Equal(t, "Something", object.Value)
}
```

Seems super valuable, right? Well, not really. Let's see what it's like to rewrite those with the "testing" package:

```
// assert equality
if 123 != 123 {
    t.Errorf("they should be equal")
}

// assert inequality
if 123 == 456 {
    t.Errorf("they should not be equal")
}

// assert for nil (good for errors)
if object == nil {
    t.Errorf("...")
}

// assert for not nil (good when you expect something)
// (and make further assertions without causing errors, because multi-clause conditionals are a thing)
if object != nil && object.Value != "Something" {
    t.Errorf("...")
}
```

Wow, we sure saved ourselves there from the dreaded _if statement_! Ok, maybe that's a bit too much sarcasm. 🙂

As you can see, ordinary Go code works just fine. Detractors will say, "it turns 1 line of code into 3" - and they're right! I would say that's a fair trade. Because what we gain in return is:

- Very clear understanding of when we t.Error vs t.Fatal.
- Very clear understanding of what our error messages will be; and the ability to customise them.
- A tight, and very low bound on the complexity of our test assertions. It's just ordinary Go code!
- Readers don't have to have knowledge of your testing framework to read the code. They just need to understand Go. It's just ordinary Go code!

As you consider these advantages of using ordinary Go code over assertion libraries consider who your code is for. Are you writing personal code, for yourself, that nobody will ever look at? Probably not: more than likely you're writing code for a team, or a company; it will be maintained for a while, maybe even after your tenure.

Remember: most code gets read far more than the time it took to write it. Optimise for the reader, not the writer. ([1](https://www.goodreads.com/quotes/835238-indeed-the-ratio-of-time-spent-reading-versus-writing-is), [2](https://devblogs.microsoft.com/oldnewthing/20070406-00/?p=27343))

# Random other musings on the subject

Two quick notes I wanted to pick out of my emails, but which I couldn't fit anywhere above:

- Ruby is another good case study, btw: first you have to learn the language, then you have to learn the craaaaazy amount of magic in rails, activerecord, and the testing libraries - surprisingly little of which overlaps. It is shockingly slow to get going from scratch in ruby because of the many layers of magic. Building a team or a company around languages (and in this case, libraries) which dramatically reduce productivity just make no sense to me.

- We get into some real contentious space here, but just to throw it out there: DRY is not always a great north star, and has many times been shown to be unhelpful. Don't optimise your codebase for one line zingers. Optimise them for someone 5 years from now with no context coming in and having to debug an issue.

# Conclusion

Don't use assertion libraries. You don't need them. You never have. You've been optimising your codebase for 1 line zingers but making it less readable and more incomprehensible, which slows down debugging and slows down future maintainers.

Use "testing". It has all you need.

Use "cmp" and other comparison libraries if you need to do more complex comparisons (on structs, for example).

Don't use assertion libraries.
