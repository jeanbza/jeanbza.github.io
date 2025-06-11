---
layout: post
title:  "Storing protobuf-generated Go code without a registry"
date:   2025-06-10 02:55:23 -0600
categories: 
toc: true
---

Before diving into this topic, you may want to familiarize yourself with [Using Go Modules](https://go.dev/blog/using-go-modules) and subsequent posts, or [the module spec](https://go.dev/ref/mod). Go modules work considerably differently than other languages' dependency management schemes: of particular note to this article is that Go modules are comprised of the code that lives in VCS, as opposed to external registries like Artifactory, pypi, npmjs.org, and so on.

## Local protos

The following covers local protos that you own.

### That you expect to be imported

If you expect others to import your proto, declare `option go_package` and generate Go bindings to that location.

This defines the single source of truth for Go generated code for your proto, making it easy for others to use your proto without needing to generate and store the Go bindings themselves.

For example:

```sh
$ head protos/server/server.proto
syntax = "proto3";

package server;

option go_package = "github.com/username/myrepo/protos/server";
$ tree
.
|____protos
| |____server
| | |____server.pb.go
| | |____server.proto
```

Note: Public directories and releasing a tag. Remember that these Go bindings are intended to be import-able, so they should be in a non-`internal/` directory. And, if your repo has tagged releases, remember to tag a new release when you generate for the first time, so that others can begin depending on your proto-generated code.

### That you don't expect to be imported

Even if you don't expect others to import your proto, it's still best to treat it as if it will be imported.

But, if you're certain you'll never want anybody else to import it, give it an `option go_package` that generates to an `internal/` directory in your Go module and add a comment above explaining that it is not meant to be depended upon.

For example:

```sh
$ head protos/server/server.proto
syntax = "proto3";

package server;

// This proto is not intended to be depended upon, and as such always generates
// into an internal/ directory.
option go_package = "github.com/username/myrepo/internal/protogen/server";
$ tree
.
|____protos
| |____server
| | |____server.proto
|____internal
| |____protogen
| | |____server
| | | |____server.pb.go
```

## Foreign protos

The following covers foreign protos that you don't own.

WARNING: Never copy `.protos`. What follows provides nuance about whether or not to store Go bindings for a proto in your module. However, you should never copy and store a foreign `.proto` in your repo.

### With a single source of truth

If a proto declares `option go_package` and provides its Go bindings at that location, that is the single source of truth for its generated Go code and you should import the single source of truth instead of generating and storing your own Go bindings.

If you are generating with `protoc`, that means you should not provide `--go_opt=M` for the proto.

If you are generating with `buf`, that means you should not include the proto in `managed.override`.

#### Looking at a concrete example

Here's a concrete example of that:

- [`github.com/googleapis/google/cloud/speech/v1/cloud_speech.proto`](https://github.com/googleapis/googleapis/blob/c759e924aa786f3df0e64499daf97d46a27edb31/google/cloud/speech/v1/cloud_speech.proto) is a proto that defines `option go_package = "cloud.google.com/go/speech/apiv1/speechpb;speechpb";`.
- Accordingly, its generated Go code exists at [`github.com/google-cloud-go/speech/apiv1/speechpb`](https://github.com/googleapis/google-cloud-go/tree/main/speech/apiv1/speechpb).
  - Note: `cloud.google.com/go` is an alias for `github.com/google/google-cloud-go`.
- If you want to use it,
  - ✅ You should import it as `speechpb cloud.google.com/go/speech/apiv1/speechpb`.
  - ❌ You should not generate (or store) `cloud_speech.pb.go`.

#### What if a proto declares option go_package, but I can't import it?

The proto is incorrectly configured. You should reach out to the proto owners to have them either remove the option go_package or generate the Go bindings at the expected location.

### Without a single source of truth

If a foreign proto does not declare `option go_package`, you have two options:

1. If you can: convince the maintainers to add option go_package, generate their Go bindings at that location, and make it available in a Go module. This is the best option.

    However, this can be a big ask for non-Go teams that aren't used to maintaining any Go code.And, if it's a widely used proto there may need to be considerable thought given to how to migrate all existing dependers. So, it may be more pragmatic to:

2. Otherwise: Generate and store their Go bindings in your module. This is risky, since it opens you and others up to the runtime panic described in Proto global registry.

    To mitigate that, store the Go bindings an `internal/` directory. If any of your code depends on these Go bindings and is accessible to other modules (is not in `main` or `_test` package), then it should also be in an `internal/` directory. See the discussion on transitive dependencies in Proto global registry.

Doing so won't reduce your chance of running into this issue, but it does prevent anyone else accidentally depending on your copy of the Go bindings.

## Proto global registry

All protocol buffers declarations linked into a Go binary are inserted into an [in-memory global registry](https://protobuf.dev/reference/go/faq/#namespace-conflict). If two protobuf declarations linked into a Go binary have the same name, then this leads to a namespace conflict. Worse, this error is a runtime error, making it easy to find its way into a deployment.
