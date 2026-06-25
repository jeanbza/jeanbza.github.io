---
layout: post
title:  "Stubbing gRPC in Go"
date:   2020-10-08 15:55:23 -0600
categories: 
toc: true
---

# Introduction

I've several times been met with the question, "How do you stub gRPC in Go"?

This is a short blog post about how to do that.

# Mocking

Mocking gRPC clients is _super easy_ and doesn't require a separate mocking
library. Given some proto with service,

```proto
service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}
```

This will generate a `greeter.pb.go` with,

```go
type GreeterClient interface {
    SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}
```

That interface is all we need to mock away. Our code that communicates with
`Greeter` will take the `GreeterClient` interface, per usual dependency
injection. In our real code (`main.go`, etc) we'll give it a real client,
and our tests we'll give it a fake client:

```go
// Code that uses GreeterClient

type SomeService struct {
    gc greeter.GreeterClient
}
// Inject me with either a real GreeterClient or a fake one!
func NewSomeService(gc greeter.GreeterClient) *SomeService {
    return &SomeService{gc: gc}
}
```

Here's our real code:

```go
var greeterAddr = flag.String("greeterAddr", "", "--greeterAddr=greeter:12345")

func main() {
    flag.Parse()
    conn, err := grpc.Dial(*greeterAddr)
    if err != nil {
        // Handle err.
    }
    defer conn.Close()
    gc := greeter.NewGreeterClient(conn)
    serv := someservice.NewSomeService(gc)
    _ = serv // Use SomeService with real GreeterClient.
}
```

And, here's our test version using a fake GreeterClient for testing:

```go
type fakeGreeterClient struct {
    sayHelloFn func(context.Context, *HelloRequest, ...grpc.CallOption) (*HelloReply, error)
}

func (gc *fakeGreeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*greeter.HelloReply, error) {
    if gc.sayHelloFn != nil {
        return gc.sayHelloFn(ctx, in, opts...)
    }
    return nil, errors.New("fakeGreeterClient was not set up with a response - must set gc.sayHelloFn")
}

func TestSomeService(t *testing.T) {
    var requests []*greeter.HelloRequest
    gc := &fakeGreeterClient{}
    // Set up the fake greeter to return a canned message.
    gc.sayHelloFn = func(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
        requests = append(requests, in) // Record requests.
        return &greeter.HelloReply{Message: "hello world"}, nil
    }
    serv := NewSomeService(gc)
    _ = serv // Test serv.
    _ = requests // Assert on expected requests.
}
```

Simple! And, of course, `fakeGreeterClient` can be more or less complicated:
perhaps it always returns the same thing (less complicated), or perhaps it tries
to mimic the behavior of the real `Greeter` (more complicated).

# Stubbing

Sometimes we aren't able to use dependency injection, but we _can_ choose which
connection we're using. For example, this is the case with
[cloud.google.com/go/pubsub](https://pkg.go.dev/cloud.google.com/go/pubsub),
whose [NewClient](https://pkg.go.dev/cloud.google.com/go/pubsub#NewClient) does
not allow passing the underlying [google.golang.org/api/pubsub/v1 raw proto-generated client interface](https://pkg.go.dev/google.golang.org/api/pubsub/v1) but [_does_ allow passing in a conn](https://pkg.go.dev/google.golang.org/api/option#WithGRPCConn).

Another example where passing a mock is not good enough, and you have to rely
on some connection, is integration tests! For integration tests, you'd want your
real binary to talk to a `Greeter` running locally, which you can
stub/set up/view interactions in the test itself. You'll want this stub
`Greeter` to have some address like `localhost:12345` that you can pass to your
binary through a flag, like `--greeterAddr=localhost:12345`.

These are just two small examples, but this problem of needing stubbing
capabilities beyond a simple interface mock comes up enough that it justifies
its own section.

It turns out to be very similar to the above mocking strategy, with a few small
differences. Let's dive in! We'll use the second example - the integration test -
though the principles apply equally to any opaque gRPC client situation.

So, here's our main from before, but we're going to make it callable from our
test by sticking it in a non-main package:

```go
package main

var greeterAddr = flag.String("greeterAddr", "", "--greeterAddr=greeter:12345")

func main() {
    flag.Parse()
    myapp.Start(*greeterAddr)
}
```

```go
package myapp

// Start starts the app. It is like main, but tests can call it.
func Start(greeterAddr string) {
    conn, err := grpc.Dial(greeterAddr)
    if err != nil {
        // Handle err.
    }
    defer conn.Close()
    gc := greeter.NewGreeterClient(conn)
    serv := someservice.NewSomeService(gc)
    _ = serv // Use SomeService with real GreeterClient.
}
```

This is one of _many_ ways to start an integration. It allows us to run `main`
from our test, since `main` itself is unexported and not run-able from a test.

Sidenote: this method may seem hacky, but why? It's just sticking your main into
another, callable method. It's used by large projects [like etcd](https://github.com/etcd-io/etcd/blob/00e49d0c10bb931f596d801d7368b5e0ae539fbd/main.go) if you need further assurance. 🙂

There are other ways to test main. For example, you might spin up the binary by
executing shell using os.Cmd and the like. That's fine! The same principles
apply.

Anyways, we have a way to run `main` - how do we get `main` to talk to our stub?
We clearly can't pass in a `GreeterClient` interface. So, instead, we'll spin up
`Greeter` as an in-memory server, and pass the in-memory server address to
`Start`.

```go
import "net"
import "google.golang.org/grpc"

type FakeGreeterClient struct {
    sayHelloFn func(context.Context, *HelloRequest, ...grpc.CallOption) (*HelloReply, error)
}

func (gc *FakeGreeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*greeter.HelloReply, error) {
    if gc.sayHelloFn != nil {
        return gc.sayHelloFn(ctx, in, opts...)
    }
    return nil, errors.New("fakeGreeterClient was not set up with a response - must set gc.sayHelloFn")
}

func TestIntegration(t *testing.T) {
    ctx := context.Context()

    // Start FakeGreeterClient in an in-memory process.
    gc := &FakeGreeterClient{}
    l, err := net.Listen("tcp", "localhost:0") // IIRC 0 == "first available port"
    if err != nil {
        t.Fatal(err)
    }
    gsrv := grpc.NewServer(opts...)
    greeter.RegisterGreeterServer(gsrv, &gc)
    fakeGreeterAddr := l.Addr().String()
    go func() {
        if err := gsrv.Serve(s.l); err != nil {
            panic(err) // We're in a goroutine - we can't t.Fatal/t.Error.
        }
    }()

    myapp.Start(fakeGreeterAddr)
    // Test your app, which is now hooked up to FakeGreeterClient!
}
```

Ta-da! Very easy to start an in-memory gRPC fake.

# Bonus: actually testing main, forreal

Ok, the above might be a bit arcane if you have the usual HTTP/gRPC API. You're
now able to hook your app up to the fake in-memory server, but how do you
actually get your test to talk to _your_ app?

However you want! But, we'll walk through one example.

Imagine our app has some HTTP endpoints, and we want to send requests to them
from our test, and see that our app appropriately talks to `Greeter` when those
endpoints get hit. Well, in order to do that we need an address that our test
can send requests to. Let's make that happen!

Let's look at `main` and `Start` again:

```go
package main

var port = flag.Int("port", 8080, "the port this app will run on, ex --port=8080")
var greeterAddr = flag.String("greeterAddr", "", "--greeterAddr=greeter:12345")

func main() {
    flag.Parse()

    // When user kills this process, close the server.
    defer myapp.Start(*port, *greeterAddr).Close()

    // Wait forever, until a user kills this process.
    wg := &sync.WaitGroup{}
    wg.Add(1)
    wg.Wait()
}
```

```go
package myapp

// Start starts the app. Call Shutdown on the returned Server when done.
func Start(port int, greeterAddr string) *http.Server {
    conn, err := grpc.Dial(greeterAddr)
    if err != nil {
        // Handle err.
    }
    defer conn.Close()
    gc := greeter.NewGreeterClient(conn)
    serv := someservice.NewSomeService(gc)
    _ = serv // Use SomeService with real GreeterClient.

    http.HandleFunc("/sayhello", func(w http.ResponseWriter, r *http.Request) {
        if _, err := serv.SayHello(context.Background(), &greeter.HelloRequest{Name: "world"}); err != nil {
            http.Error(w, err.Error(), 500)
        }
    })
    
    srv := &http.Server{Addr: fmt.Sprintf(":%d", port)}
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    return srv
}
```

Mostly the same, except we now have an HTTP API. This is just an example - a
gRPC server would look similar.

Onto the integration test!

```go
import "net"
import "google.golang.org/grpc"

type FakeGreeterClient struct {
    sayHelloFn func(context.Context, *HelloRequest, ...grpc.CallOption) (*HelloReply, error)
}

func (gc *FakeGreeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*greeter.HelloReply, error) {
    if gc.sayHelloFn != nil {
        return gc.sayHelloFn(ctx, in, opts...)
    }
    return nil, errors.New("fakeGreeterClient was not set up with a response - must set gc.sayHelloFn")
}

func TestIntegration(t *testing.T) {
    ctx := context.Context()

    // Start FakeGreeterClient in an in-memory process.
    gc := &FakeGreeterClient{}
    l, err := net.Listen("tcp", "localhost:0") // IIRC 0 == "first available port"
    if err != nil {
        t.Fatal(err)
    }
    gsrv := grpc.NewServer(opts...)
    greeter.RegisterGreeterServer(gsrv, &gc)
    fakeGreeterAddr := l.Addr().String()
    go func() {
        if err := gsrv.Serve(s.l); err != nil {
            panic(err) // We're in a goroutine - we can't t.Fatal/t.Error.
        }
    }()

    myappPort := openPort(t)
    myappAddr := fmt.Sprintf("localhost:%d", myappPort)

    srv := myapp.Start(myappPort, fakeGreeterAddr)
    defer srv.Close()
    
    // Test your app, which is now hooked up to FakeGreeterClient, by sending
    // requests to myappAddr!
}

// openPort returns an open port.
func openPort(t *testing.T) int {
    t.Helper()
    l, err := net.Listen("tcp", ":0")
    defer l.Close()
    if err != nil {
        t.Fatal(err)
    }

    u, err := url.Parse(l.Addr())
    if err != nil {
        t.Fatal(err)
    }
    if u.Port() == "" {
        t.Fatalf("unable to parse a port from %s", l.Addr())
    }

    p, err := strconv.Atoi(u.Port())
    if err != nil {
        t.Fatal(err)
    }
    return p
}
```

# Conclusion

Stubbing and mocking gRPC servers in Go is very easy, doesn't require any
libraries, and obviates the need for pre-provided fakes/mocks/emulators/etc.

