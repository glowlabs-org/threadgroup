threadgroup
-----------

###### This code is from an old open source library I made at Nebulous Labs, which has since gone out of business. I wasn't sure how long the repo was going to stay online, so I made a new home for the code so that it could be used by Glow software.

threadgroup is a utility used to facilitate the coordinated shutdown of a set
of resources. Instead of individually managing the shutdown of each resource, a
single call to `tg.Stop()` will gracefully clean up everything all at once in a
coordinate way that respects resource dependencies.

Resources can be added to a ThreadGroup by calling `tg.Add()`. They can then
use the ThreadGroup to watch for a stop signal to determine that all resources
related to the ThreadGroup are being shut down. A resource signals to the
ThreadGroup that it has completed shutdown by calling `tg.Done()`.

If a resource needs cleanup code to run when it gets shut down, it can schedule
cleanup code by calling `tg.OnStop()` and `tg.AfterStop()`. All functions added
with `tg.OnStop()` will be called after the ThreadGroup has stopped, but
without blocking to wait until all of the resources have signaled that shutdown
is complete. All functions added with `tg.AfterStop()` will only be called
after all resources connected to the ThreadGroup have signaled that they have
completed shutdown.

A `net.Listener` for example can use `tg.OnStop()` to have the ThreadGroup call
`listener.Close()`. A logger on the other hand will probably handle its own
shutdown in `tg.AfterStop()`, because the logger will not want to close itself
until after all other functions have had a chance to log any errors they
encountered during shutdown.

The ThreadGroup will call all `OnStop` and `AfterStop` functions in reverse
order to how they were added, and the ThreadGroup will block on each one before
calling the next one. Every function will be called regardless of any errors
that are returned. Once every function in the `OnStop` and `AfterStop` queue
has been called, `tg.Stop()` will return a single error that contains all of
the errors produced during shutdown.

Long running background threads should be created by calling `tg.Launch()`.
This will add the thread to the ThreadGroup before creating the thread, and
will schedule a call to `tg.Done()` when the thread returns. Launch will return
an error if Launch() is called after the ThreadGroup has already been stopped.

The ThreadGroup offers two helper functions for performing long sleeps. There's
`tg.Sleep()`, which will sleep like `time.Sleep` but will exit early when
`tg.Stop()` is called, and there is `tg.AfterFunc()` which will call a function
after an amount of time has elapsed like `time.AfterFunc()`, except that it
will exit early without calling the function if `tg.Stop()` is called.

Finally, the ThreadGroup offers a `StopChan()` function and an `IsStopped()`
function. `tg.StopChan()` returns a channel which will be closed when
`tg.Stop()` is called. `IsStoppped()` will immediately return true or false
depending on whether `tg.Stop()` has been called yet.

Example:
```go
var tg threadgroup.ThreadGroup

// Create a logger and set it to shutdown upon closing. Because log.Close() is
// the first call added to AfterStop, it will be the last function called when the
// ThreadGroup shuts down.
log := NewLogger()
tg.AfterStop(func() error {
	return log.Close()
})

// Create a background thread that will call `net.Dial()` once an hour. Use the
// ThreadGroup to ensure that the Dial call will be canceled upon shutdown, and
// also to ensure that the hour long sleeps between calls will be interrupted.
func InfrequentDial() {
	// Repeatedly perform a dial. Latency means the dial could take up to a
	// minute, which would delay shutdown without a cancel chan.
	for {
		// Perform the dial, but abort quickly if 'Stop' is called.
		dialer := &net.Dialer{
			Cancel:  tg.StopChan(), // This will cause the Dial to be cancelled
			Timeout: time.Minute,
		}
		conn, err := dialer.Dial("tcp", 8.8.8.8)
		if err == nil {
			conn.Close()
		}

        // Sleep for an hour, using the ThreadGroup's sleep to exit early if
        // tg.Stop() is called.
        if tg.Sleep(time.Hour) {
            return
        }
	}

    // Log that the dial thread has shut down cleanly. The logger does not
    // close until the `AfterStop()` functions are called in the threadgroup, and
    // those functions will be blocked until this thread has exited because this
    // thread was created using `tg.Launch()`. Therefore, the logger is guaranteed
    // to still be working at the moment this line is called, even though it is called
    // during shutdown.
	log.Println("InfrequentDial closed cleanly")
}()
err = tg.Launch(InfrequentDial)
if err != nil {
    fmt.Println("ThreadGroup was stopped before the InfrequentDial thread could be launched")
}

// Create a background thread that will listen for new connections but also
shut down cleanly when tg.Stop() is called.
func LongListener() {
	// Create the listener.
	listener, err := net.Listen("tcp", ":12345")
	if err != nil {
		return
	}
    // Close the listener as soon as 'Stop' is called. OnStop is used here,
    // because the listener should be closed immediately and does not need to wait for
    // other resources to finish shutting down first.
    //
    // In this case, OnStop is actually necessary, because the OnStop call here is what
    // will cause the infinite loop in the listener to break. Using AfterStop would be a
    // mistake because the infinite loop would not break until AfterStop is called,
    // and AfterStop would not be called until this thread returns, which would block
    // until listener.Close is called, resulting in deadlock.
	tg.OnStop(func() error {
		return listener.Close()
	})

	for {
		conn, err := listener.Accept()
		if err != nil {
            // Accept will return an error as soon as the listener is closed,
            // which will happen immediately after tg.Stop() is called.
			return
		}
		conn.Close()
	}

    log.Println("LongListener closed cleanly")
}()
err = tg.Launch(LongListener)
if err != nil {
    fmt.Println("ThreadGroup was stopped before the LongListener could be launched")
}

// Calling Stop will result in a quick, organized shutdown that closes all
// long-running resources.
err := tg.Stop()
if err != nil {
	fmt.Println(err)
}
```
