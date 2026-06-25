---
layout: post
title:  "Complex assertions"
date:   2022-02-08 15:55:23 -0600
categories:
toc: true
---

This is part of a series of posts on testing in Go:

- [Stubbing gRPC clients in Go tests](/2020/10/08/stubbing-grpc-clients)
- [testing in Go](/2022/02/08/testing-in-go)
- **Complex assertions**

I've seen a lot of complexity in custom assertions. That tends to slow teams down and increase the cost of ownership for many applications.

I thought I'd briefly showcase that.

The following are real code examples pulled from real codebases.

## Custom matcher examples

Consider the following Java code:

```java
assertParagraphElement(bodyStructuralElements
    .get(25)
    .getParagraph()
    .getElements()
    .get(0), 311, 312, Person.class);
```

Here's another in C++:

```cpp
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

I couldn't really tell you what's going on in either without looking at `assertParagraphElement` or `dos_quotas::HasRequest`.

## Custom matchers tend to reduce comprehensibility of your tests

I tend to see people using custom matchers either because it's super terse, or as a way to implement a helper function.

But, custom matchers often come with a sharp drop in comprehensibility of your tests. Readers have to guess functionality based on the name and docs. And in my experience, people writing custom matchers for their application or project (as opposed to for a general-use library) tend not to put too much effort in making their matchers well named and documented.

That means that readers will often need to look at the customer matcher's code to grok what's going on, and that tends to be hard since matchers are very often written with generics, reflection, or macros.

This comprehensibility cost is high and very under-appreciated: it slows down reading (and extending) your tests and makes it harder to onboard new people to your part of the codebase.

I'd recommend preferring more verbose but easier to read code to keep to keep your team going fast. When you need a helper, I'd suggest writing it as a simple function rather than as a matcher.