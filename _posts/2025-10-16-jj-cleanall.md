---
layout: post
title:  "A jj script to lint your entire graph"
date:   2025-10-06 02:55:23 -0600
categories: 
toc: true
---

This is the second of a series of posts about `jj`. The first is
[A jj plugin for bash-it](/2025/10/06/jj-bash-it.html).

-----------------

At Netflix, we use [the spotless linter](https://github.com/diffplug/spotless).
The main gripe I have with it is that it's not integrated with any of our IDEs.
In contrast, in Go, `gofmt` is integrated with all my IDEs and linting happens
as I program; with spotless I have to remember to run `./gradlew spotlessApply`
before I push my changes, or else my PR's build will break when it gets to the
lint check step.

When I'm working with `jj` rev chains, it's annoying to
`jj edit <rev> && ./gradlew spotlessApply` each and every rev before sending
them all off to be CI/CD'd with `jj git push --all`.

So, I made this little script. I hope you'll find it useful too:

```sh
#!/bin/bash
set -e
for i in `jj log -r "mutable()" --no-graph -T 'change_id ++ "\n"' | tail -r`; do
  jj edit $i
  ./gradlew spotlessApply
  ./gradlew spotbugsMain
done
jj log
```

Of course, sub out the for loop with whatever you want to do at each rev.

It makes use of `jj`'s
[fantastic templating language](https://jj-vcs.github.io/jj/latest/templates/)
to walk the graph from oldgest to newest change, linting as it goes.
