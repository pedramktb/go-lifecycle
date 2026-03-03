# Lifecycle

Lifecycle is a go package that provides graceful shutdown for go (sub-)programs.

## Behavior
Context returns a context with a lifecycle inside, a cancel function, and an error channel.
ContextFrom does the same, but lets you provide a parent context.
This lifecycle inside context is used to close all closers to finish before exiting a (sub-)program.
The lifecycle holds a map of closing groups. By using RegisterCloser you can register a new closer with a group to be called when the context is cancelled.
Each group calls its closers in a separate goroutine and in reverse order (similar to defer).
The wait is limited to the given timeout or a terminate or an interrupt signal after the initial
cancelation.
This means closing a running program immediately from shell, requires 2 interrupts.
The additional cancel function can be used the start the shutdown process from inside the program.
The returning channel should be used to prevent the main from exiting and receive the shutdown errors.
This channel is closed when all closers complete and all errors are received or a preemptive shutdown is triggered.
In case of a preemptive shutdown, ErrShutdownTimeout or ErrForcedShutdown will be added to the channel before closing the channel.

## Example
```go
package main

import (
    "github.com/pedramktb/go-lifecycle"
    "time"
    "your-module/database"
    "your-module/cache"
    // other imports...
)

func main() {
    ctx, cancelFunc, shutdownErrsChan := lifecycle.Context(time.Minute)
    defer cancelFunc()

    // If you need to derive from an existing parent context, use:
    // parent := context.Background()
    // ctx, cancelFunc, shutdownErrsChan := lifecycle.ContextFrom(parent, time.Minute)

    // ...
    db := database.New(ctx)
    err := lifecycle.RegisterCloser(ctx, db.Close, "db-stack")
    if err != nil {
        // ...
    }
    cache := cache.New(ctx)
    // registering cache on the same group as db makes sure cache is flushed to the db before starting to close the db
    err = lifecycle.RegisterCloser(ctx, cache.Close, "db-stack")
    if err != nil {
        // ...
    }
    // ...

    for err := range shutdownErrsChan {
        errors = append(errors, err)
    }
    if len(errors) > 0 {
        logger.Error("one or more modules failed to shutdown properly", errors)
    }
}
```
