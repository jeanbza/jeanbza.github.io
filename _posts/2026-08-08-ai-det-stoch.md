---
layout: post
title:  "LSCs, AI agents, and determinism"
date:   2026-08-08 02:55:23 -0600
categories: 
toc: true
---

Large scale code changes (LSCs) are groups of code changes that all try to
accomplish the same goal, across a huge number of codebases.

For example, a common LSC is to migrate all codebases at a company from
libraryX@v1 to libraryX@v2, changing code along the way to account for
differences in API between v1 and v2. Due to their breadth, LSCs have high
cognition cost, owing to the high amount of contexts you have to reason about.
But also due to their breadth, they tend to have bounded problem spaces: the
wider a change has to be applied, the more generic the change must be. A change
that can't be generically expressed is not an LSC, but instead a series of
individual changes.

AI agents are one way to attempt to tackle LSCs. Goals can be expressed
generically and handed to AIs as prompts to work on all target codebases. But,
when AI agents are set loose to perform LSCs, LSCs' high cognition cost
compounds with AI non-determinism to result in low confidence in the changes
being applied, incur high costs, and produce results which are unauditable.

This article presents a different approach, based on a simple observation: most
changes that can be expressed generically can also be expressed mechanically. AI
agents explore and solve the LSC's problem space in an evaluation flywheel, and
encode their results in a metaprogramming program which _itself_ is used to
perform the LSC in a way that is deterministic, leads to higher confidence, is
cheap, and is as auditable as ordinary source code.

![Explore](/assets/agent_exploration.gif)

## Context

At Google and now at Netflix, I find myself frequently having to perform large
scale changes to code: changes to hundreds or thousands of codebases. For
example, at Google, when I rewrote the internal Go `status` library ([similar
external version
here](https://pkg.go.dev/google.golang.org/grpc/internal/status)) to support [Go
1.13 error wrapping/unwrapping](https://go.dev/blog/go1.13-errors), Google's
internal one-version-of-every-library + monorepo setup meant that I had to also
go make this work for ~56k Go programs. That involved making changes to hundreds
of programs with non-conforming / Hyrum's-Law type oddities at the same time.

Most recently at Netflix, I've been working on problems requiring LSCs over our
thousands of Go projects. An example recent, long-running LSC is periodic
remediation of vulnerable dependencies.

Doing this kind of work in 2026 is dramatically faster with AI Agents, but
surprisingly there's a good lesson to be learned from the old way of doing it
which can be paired with AI agents.

Before AI agents, we used to perform these changes by hand, and quickly learned
instead to build metaprogramming programs: programs which modified programs.

With AI agents, it's natural to think to replace this with hordes of AI
sub-agents performing code change on each repository, with some shared prompt or
goal. However, it turns out to still be more advantageous to write a program for
the LSC, albeit now with an AI agent to explore the problem space and encode the
solution into that program.

The end result is a small program which, when run on all the target
repositories, solves the LSC goal in a deterministic, cheap, and auditable
manner.

Let's take a look in closer detail.

## Evaluation framework

This technique starts with an evaluation framework and a discovery / actuate /
evaluate loop. Using the vulnerability remediation project above, the evaluation
framework is largely self-evident. The goal is to remediate all Go repositories
of vulnerable dependencies. For each repository,

- Go's [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
reports whether there are any vulnerabilities. (Goal: 0 fixable vulnerabilities)
- The output and exit code of whatever program we write tells us whether we were
successful, and if not what failed. (Goal: no failures)
- A PR's CI/CD log tells us whether a PR to perform that remediation is valid,
and if not what failed. (Goal: valid PR)

This is all we need to begin our loop.

## Discovery loop

With a sufficiently large enough set of repositories (which we have at Netflix,
or is generally available in GitHub), and enough compute, and enough tokens, and
a bounded problem space, we can solve the problem space and produce a single
solution.

We produce a solution to the problem space in the form of a program. That
program remediates vulnerabilities in whatever repository it is run in.

We encode the problem space, as we discover it, as a series of [txtar
tests](https://pkg.go.dev/golang.org/x/tools/txtar). That becomes the early
signal in our evaluation framework; it doubles up as a regression test
mechanism; and it triples up as audit documentation (This is how the program is
built to behave under scenario X, Y, Z).

As the agent explores the space, it encodes new situations it finds as txtar
tests, writes code to solve the expanded problem scope, verifies it first
against the txtar test and then by attempting to send PRs to its test repository
cohort, looking at failure and build logs, and re-assessing. When it reaches
quiescence, it expands its test cohort exponentially.

When we applied this strategy to the vulnerability remediation problem, encoding
the problem space as txtar texts, we found fewer and fewer new cases as we
expanded the test pool:

![Discovery curve](/assets/discovery_curve.jpg)

## Deterministic solution

The end result is a single program which can be run on a repository to
deterministically upgrade dependencies to remediate vulnerabilities. And since
it's a program and not an LLM, we also benefit being able to audit its behaviour
through normal source control history. Both the determinism and auditability
increase confidence in its use in large scale changes considerably.

The other major benefit is cost and time: the program runs in <2s, and costs
$0+a bit of CPU.

I ran a benchmark against 50 repositories, using the program in one test and an
AI agent (Claude Sonnet 5, medium effort) in the other set, both tasked with
remediating vulnerabilities. On average, the AI agent took ~50k[^1] tokens
($0.15/run) and 4m[^2]/run. Across ~2000 of Go repos, that's $750/run. We run
this large scale change every 6h to remediate vulnerabilities quickly: that
means we'd pay ~$0.66M/yr. At Google's scale with 30x-40x those Go repos, that's
more like $20M/yr.

![AI cost for LSCs](/assets/ai_agent_vs_program_cost.jpg)

And, that's a minimum: as developers 10x their velocity, they also 10x the repos
they create, which 10x this cost. The program by comparison is ~$0 and will
forever stay that way.

## Long tail

There may be a long tail of issues to deal with. In the cases we've dealt with
so far, this has not been the case, but it's fair to imagine that there might
be.

Of course, two simple solutions exist:

- Run an AI agent as a backstop.
- Keep the discovery loop infrastructure and allow long-tail findings to
continually improve the program.

## A better solution

It's attractive to throw AI Agents at every problem, since they're so fast at
exploring problems at producing solutions. But in the case of LSCs, the way that
you apply AI Agents makes a difference.

At Netflix we've had good success marrying the speed and ability of AI agents to
explore the unknown, along with the observation that most LSCs have bounded
problem spaces whose solutions are generically and mechanically expressable, to
have AI agents create LSC programs.

The result is a solution that runs in seconds instead of hours, costs nothing
instead of hundreds of thousands a year, behaves the same way on the nth
repository as it did on the first, and can be reviewed the way we review
everything else: by reading the source and the logs.

[^1]: 50k is the rough median. Low end was 28k, high end was 77k. Most were in the 40k-60k range.

[^2]: 4m is the rough median. Low end was 2m16s, high end was 8m53s.
