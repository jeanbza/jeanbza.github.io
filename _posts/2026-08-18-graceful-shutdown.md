---
layout: post
title:  "Graceful shutdown with HTTP & gRPC in Go"
date:   2026-08-08 02:55:23 -0600
categories: [go, kubernetes, grpc]
description: "A quick example of how to do graceful shutdown of your gRPC and HTTP server in Go, particularly useful on Kubernetes."
toc: true
---

At Netflix, we run our containers on Kubernetes. Sometimes Kubernetes has to
terminate your container, for any number of reasons: a new version is being
deployed, or your container is being migrated elsewhere, or a scale down is
happening, etc. As the [Kubernetes Pod Lifecycle docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
describes, the application in your container will be sent a SIGTERM/SIGINT, and
then Kubernetes will for your application to gracefully shut down, and if it
hasn't done so after a small grace period, Kubernetes will SIGKILL it.

So, I often find myself starting applications by writing graceful shutdown
logic. And since most of the application programming that I do uses HTTP and
gRPC servers, I end up writing graceful shutdown for those fairly frequently.

I thought I'd codify that here for myself, and for anyone else interested in
seeing how to gracefully shut down your gRPC and HTTP servers in Go.

The key parts are:

1. We use `signal.NotifyContext` to attach SIGINT/SIGTERM to context
   cancellation.
1. We then feed that context to an `errgroup.WithContext` to create context
cancellation (SIGTERM/SIGINT) aware goroutines.
1. We spawn 2 goroutines to serve HTTP/gRPC, respectively.
1. We spawn 2 more HTTP/gRPC goroutines to watch for context cancellation
(SIGTERM/SIGINT) and begin graceful shutdown.
1. We bound graceful shutdown to a timeout, after which we forcefully shut down.
    - This part is take it or leave it: Kubernetes will send SIGKILL after
    whatever grace period it allows. So, you could just use that mechanism. I
    personally prefer to own shutdown as much as I can, though, so that I can
    make sure to call all the logs/metrics/tracing flushing.

```go
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"golang.org/x/sync/errgroup"
	"google.golang.org/grpc"

	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

var httpPort = flag.Int("httpPort", 7000, "The port to serve HTTP on")
var grpcPort = flag.Int("grpcPort", 8000, "The port to server gRPC on")

const httpShutdownTimeout = 5 * time.Second // How long HTTP shutdown is allowed to take.
const grpcShutdownTimeout = 5 * time.Second // How long gRPC shutdown is allowed to take.

func main() {
	flag.Parse()

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	if err := run(ctx); err != nil {
		slog.Error("application exited with 1", "error", err)
		// Make sure to flush logs/metrics/traces/etc by this point! And keep in
		// mind that defer doesn't run when you os.Exit(1), so you might need
		// to invoke them manually depending on where you performed init.
		os.Exit(1)
	}
	slog.Info("application exited with 0")
}

func run(ctx context.Context) error {
	// Start with port binding.
	httpListener, err := net.Listen("tcp", fmt.Sprintf(":%d", *httpPort))
	if err != nil {
		return fmt.Errorf("http listen on port %d failed: %v", *httpPort, err)
	}
	defer httpListener.Close()

	grpcServerListener, err := net.Listen("tcp", fmt.Sprintf(":%d", *grpcPort))
	if err != nil {
		return fmt.Errorf("grpc listen on port %d failed: %v", *grpcPort, err)
	}
	defer grpcServerListener.Close()

	eg, gCtx := errgroup.WithContext(ctx)

	// HTTP server.
	mux := http.NewServeMux()
	mux.HandleFunc("/foo", func(w http.ResponseWriter, _ *http.Request) {
		// An example HTTP endpoint. But you might implement something else.
		if _, err := fmt.Fprintln(w, "bar"); err != nil {
			slog.Error("/foo failed during write", "error", err)
		}
	})
	httpServer := &http.Server{Handler: mux}

	// Start HTTP serve and shutdown goroutines.
	eg.Go(func() error { // HTTP serve.
		slog.Info(fmt.Sprintf("HTTP server starting at port :%d", *httpPort))
		if err := httpServer.Serve(httpListener); err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("HTTP server failed", "error", err)
			return err
		}
		slog.Info("HTTP server shut down")
		return nil
	})
	eg.Go(func() error { // HTTP graceful -> forceful shutdown.
		<-gCtx.Done()
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), httpShutdownTimeout)
		defer shutdownCancel()
		slog.Info("HTTP server gracefully shutting down")
		if err := httpServer.Shutdown(shutdownCtx); err != nil {
			slog.Error("HTTP server shutdown failed. forcefully shutting down")
			return errors.Join(err, httpServer.Close())
		}
		return nil
	})

	// gRPC server.
	grpcServer := grpc.NewServer()
	pb.RegisterGreeterServer(grpcServer, &fooService{})

	// Start gRPC serve and shutdown goroutines.
	eg.Go(func() error { // gRPC serve.
		slog.Info(fmt.Sprintf("gRPC server starting at port :%d", *grpcPort))
		if err := grpcServer.Serve(grpcServerListener); err != nil {
			slog.Error("gRPC server failed", "error", err)
			return err
		}
		slog.Info("gRPC server shut down")
		return nil
	})
	eg.Go(func() error { // gRPC graceful -> forceful shutdown.
		<-gCtx.Done()
		stopped := make(chan struct{})
		go func() {
			slog.Info("gRPC server gracefully shutting down")
			grpcServer.GracefulStop()
			close(stopped)
		}()
		select {
		case <-stopped:
		case <-time.After(grpcShutdownTimeout):
			slog.Error("gRPC server forcefully shutting down")
			grpcServer.Stop()
			return fmt.Errorf("gRPC server was unable to gracefully shut down")
		}
		return nil
	})

	// This returns nil if shutdown was successful, or the first err for any
	// shutdown issue.
	return eg.Wait()
}

type fooService struct {
	pb.UnimplementedGreeterServer
}
```

Let's run it and send it a SIGINT:

```sh
$ go run .
2026/08/18 09:19:39 INFO HTTP server starting at port :7000
2026/08/18 09:19:39 INFO gRPC server starting at port :8000
^C2026/08/18 09:19:40 INFO gRPC server gracefully shutting down
2026/08/18 09:19:40 INFO HTTP server gracefully shutting down
2026/08/18 09:19:40 INFO HTTP server shut down
2026/08/18 09:19:40 INFO gRPC server shut down
2026/08/18 09:19:40 INFO application exited with 0
```