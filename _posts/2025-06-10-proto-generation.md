---
layout: post
title:  "Generating protobuf-generated Go code without a registry"
date:   2025-06-10 01:55:23 -0600
categories: 
toc: true
---

I've recently had to generate Go code from protos, without the aid of a proto
registry. This article covers my advice for anyone needing to do the same.

## Generating Go code with protoc

Generating code with `protoc` is best for protos that have few or no
dependencies.

### Simple code generation

Imagine two simple protos:

```pb
// protos/user/user.proto
syntax = "proto3";

package user;

option go_package = "github.com/username/myrepo/protos/user";

message User {
    string name = 1;
}
```

```pb
// protos/server/server.proto
syntax = "proto3";

package server;

option go_package = "github.com/username/myrepo/protos/server";

import "user/user.proto";

message Server {
    repeated user.User users = 1;
}
```

Generate Go code (.pb.gos) from this simple proto with `protoc`:

```sh
$ protoc protos/**/*.proto \
    --go_out=protos \
    -I protos/ \
    --go_opt=paths=source_relative
$ tree
.
|____protos
| |____server
| | |____server.pb.go
| | |____server.proto
| |____user
| | |____user.pb.go
| | |____user.proto
```

Warning: This approach allows others to depend on your generated Go code.

This example signs you up to maintaining generated Go code at `github.com/username/myrepo/protos/server` and `[...]/protos/user`. Whether or not this is a good idea is discussed in "Storing generated code". For now suffice it to say that instead of declaring `option go_package` you could instead provide `--go_opt=M<proto>=<go import>`. And, instead of generating your `.pb.go`s into the publicly import-able `protos/`, you could generate them into an `internal/` directory which is not publicly import-able.

If you need to produce gRPC language bindings, add the gRPC option equivalents `--go-grpc_out` and `--go_grpc_opt`:

```sh
$ protoc protos/**/*.proto \
    --go_out=protos \
    --go-grpc_out=protos \
    -I protos/ \
    --go_opt=paths=source_relative \
    --go-grpc_opt=paths=source_relative
```

### Foreign protos

Part of your dependency graph may include protos in other repos:

```proto
// protos/user/user.proto
syntax = "proto3";

package user;

option go_package = "github.com/username/myrepo/protos/user";

// Not in this repo. Also not repo-rooted, which we'll discuss below. We could
// change this if we owned user.proto, but if this occurred in a foreign proto
// that we can't change then we'd have to work around it.
import "non/repo/rooted/import.proto";

message User {
    string name = 1;
}
```

You'll need to git clone the foreign protos and include them with `-I` during your `protoc` invocation.

You'll may also need to account for quirks, such as:

- Dealing with imports not rooted at its repo root. Consider your code A importing B, and B importing another proto C, but not at C's repo root. Instead, B imports C at at `$REPO-ROOT/some/nested/dir`. Since you don't own B (another team / company does), you can't change B. Get around this with `-I` rooted at the place the import expects, as shown below.

- Dealing with imports that do not define `option go_package`. When protos in your dependency graph don't define `option go_package`, you'll have to tell `protoc` which go_package to use with `--go_opt=M<proto>=<go import path>`. You'll probably have to generate the foreign proto's `.pb.go` into your module and point the aforementioned Go import path there, under the assumption that if they haven't defined an option go_package then it's also likely they are not generating and publishing Go bindings for you to use.

```sh
TMP=$(mktemp -d)
git clone https://github.com/foreignorg/foreignrepo.git "$TMP/foreign"
mkdir -p internal/protogen
protoc protos/**/*.proto \
    non/repo/rooted/import.proto \
    --go_out=internal/protogen \
    -I protos \
    --go_opt=Muser/user.proto=github.com/username/myrepo/internal/protogen/user \
    --go_opt=Mserver/server.proto=github.com/username/myrepo/internal/protogen/server \
    -I $TMP/path/to/expected/import/root \
    --go_opt=Mnon/repo/rooted/import.proto=github.com/foreignorg/foreignrepo/path/to/expected/import/root \
    --go_opt=paths=source_relative
```

Which when run would generate:

```sh
$ tree
.
|____internal
| |____protogen
| | |____server
| | | |____server.pb.go
| | |____user
| | | |____user.pb.go
| | |____non
| | | |____repo
| | | | |____rooted
| | | | | |____import.pb.go
```

Note that all the generated code was generated into `internal/`. That involved overwriting `user.proto` and `server.proto`'s `option go_package` with `--go_opt=M`. That's because different generated versions of `import.proto` may not exist (directly or transitively) in any Go import graph due to Go's global proto registry. So, `internal/` is used to prevent anybody from depending on these `.pb.go`s. Read more on this in "Storing generated code".

### Large dependency graphs

Using `protoc` becomes progressively more unwieldy the larger your proto dependency graph becomes, since you have to specify all transitive dependencies at `protoc` time.

Let's now look at a better tool for managing larger dependency graphs.

## Generating Go code with buf

Generating code with `buf` is better for protos that have more than a few dependencies.

### Simple code generation

Imagine two simple protos:

```sh
// protos/user/user.proto
syntax = "proto3";

package user;

option go_package = "github.com/username/myrepo/protos/user";

message User {
    string name = 1;
}
```

```sh
// protos/server/server.proto
syntax = "proto3";

package server;

option go_package = "github.com/username/myrepo/protos/server";

import "user/user.proto";

message Server {
    repeated user.User users = 1;
}
```

Generate Go code (.pb.gos) from these protos with buf by adding:

- A `buf.yaml`:

    ```yml
    # buf.yaml
    # For details on buf.yaml configuration, visit https://buf.build/docs/configuration/v2/buf-yaml
    version: v2
    lint:
    use:
        - STANDARD
    breaking:
    use:
        - FILE
    modules:
    - path: protos/
    ```

- A `buf.gen.yaml`:

    ```yml
    # buf.gen.yaml
    # For details on buf.yaml configuration, visit https://buf.build/docs/configuration/v2/buf-gen-yaml
    version: v2
    plugins:
    - remote: buf.build/protocolbuffers/go:v1.36.6
        out: protos
        opt:
        - paths=source_relative
    ```

Generate code with `buf generate`:

```sh
$ buf generate
$ tree
.
|____protos
| |____server
| | |____server.pb.go
| | |____server.proto
| |____user
| | |____user.pb.go
| | |____user.proto
```

Warning: This approach allows others to depend on your generated Go code.

This example signs you up to maintaining generated Go code at `github.com/username/myrepo/protos/server` and `[...]/protos/user`. Whether or not this is a good idea is discussed in "Storing generated code". For now suffice it to say that instead of declaring `option go_package` you could instead provide `--go_opt=M<proto>=<go import>`. And, instead of generating your `.pb.go`s into the publicly import-able `protos/`, you could generate them into an `internal/` directory which is not publicly import-able.

If you need to produce gRPC language bindings, add the gRPC plugin to `buf.gen.yaml`:

```yml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go:v1.36.6
    out: protos
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go:v1.5.1
    out: protos
    opt:
      - paths=source_relative
```

### Foreign protos

Part of your dependency graph may include protos in other repos:

```proto
// protos/user/user.proto
syntax = "proto3";

package user;

option go_package = "github.com/username/myrepo/protos/user";

// Not in this repo. Also not repo-rooted, which we'll discuss below. We could
// change this if we owned user.proto, but if this occurred in a foreign proto
// that we can't change then we'd have to work around it.
import "non/repo/rooted/import.proto";

message User {
    string name = 1;
}
```

Foreign protos that are neither in your repo nor the Buf Schema Registry (BSR) can be imported, but in a roundabout way. You'll need to git clone them yourself and include them as modules in buf.yaml.

Before we do so, let's quickly recap proto quirks you might run into, described above in "Generating code with protoc: Foreign protos". We'll need to contend with the fact that the `import.proto` import was not rooted at its repo root, and that `import.proto` does not define `option go_package`.

```yml
# buf.yaml
version: v2
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
modules:
  - path: protos/
  - path: tmp/path/to/expected/import/root
```

The non-repo rooted import is handled by specifying the path at which the `import "non/repo/rooted/...";` import works.

To handle the lack of `option go_package`, we'll need to turn on managed mode in `buf.gen.yaml`:

```yml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go:v1.36.6
    out: internal/protogen
    opt:
      - paths=source_relative
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      path: non/repo/rooted/
      value: github.com/foreignorg/foreignrepo/path/to/expected/import/root
    - file_option: go_package_prefix
      path: user/user.proto
      value: github.com/username/myrepo/internal/protogen
    - file_option: go_package_prefix
      path: server/server.proto
      value: github.com/username/myrepo/internal/protogen
```

Next, let's perform the `git clone` and `tmp/` management:

```sh
git clone https://github.com/foreignorg/foreignrepo.git tmp/foreign
buf generate
```

Which when run generates:

```sh
$ tree
.
|____internal
| |____protogen
| | |____server
| | | |____server.pb.go
| | |____user
| | | |____user.pb.go
| | |____non
| | | |____repo
| | | | |____rooted
| | | | | |____import.pb.go
```

Note that all the generated code was generated into `internal/`. That involved overwriting `user.proto` and `server.proto`'s `option go_package` with `--go_opt=M`. That's because different generated versions of `import.proto` may not exist (directly or transitively) in any Go import graph due to Go's global proto registry. So, `internal/` is used to prevent anybody from depending on these `.pb.go`s. Read more on this in "Storing generated code".

### Concurrent git clones

If you have many `git clone` statements, consider using `&` and `wait` to run them concurrently:

```sh
git clone ... &
git clone ... &
git clone ... &
wait
buf generate
```
