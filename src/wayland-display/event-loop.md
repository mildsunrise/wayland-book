# Incorporating an event loop

libwayland provides its own event loop implementation for Wayland servers to
take advantage of, but the maintainers have acknowledged this as a design
overstep. For clients, there is no such equivalent. Regardless, the Wayland
server event loop is useful enough, even if it's out-of-scope.

## Wayland server event loop

Each `wl_display` created by libwayland-server has a corresponding
`wl_event_loop`, which you may obtain a reference to with
`wl_display_get_event_loop`. If you're writing a new Wayland compositor, you
will likely want to use this as your only event loop. You can add file
descriptors to it with `wl_event_loop_add_fd`, and timers with
`wl_event_loop_add_timer`. It also handles signals via
`wl_event_loop_add_signal`, which can be pretty convenient.

With the event loop configured to your liking to monitor all of the events your
compositor has to respond to, you can process events and dispatch Wayland
clients all at once by calling `wl_display_run`, which will process the event
loop and block until the display terminates (via `wl_display_terminate`). Most
Wayland compositors which were built from the ground-up with Wayland in mind (as
opposed to being ported from X11) use this approach.

However, it's also possible to take the wheel and incorporate the Wayland
display into your own event loop. `wl_display` uses the event loop internally
for processing clients, and you can choose to either monitor the Wayland event
loop on your own, dispatching it as necessary, or you can disregard it
entirely and manually process client updates. If you wish to allow the Wayland
event loop to look after itself and treat it as subservient to your own event
loop, you can use `wl_event_loop_get_fd` to obtain a [poll][poll]-able file
descriptor, then call `wl_event_loop_dispatch` to process events when activity
occurs on that file descriptor. You will also need to call
`wl_display_flush_clients` when you have data which needs writing to clients.

[poll]: https://pubs.opengroup.org/onlinepubs/009695399/functions/poll.html

## Wayland client event loop

libwayland-client, on the other hand, implements a small event loop for simple
programs where activity on one `wl_display` is the only event source. In that
case, this simple loop will suffice:

```c
while (wl_display_dispatch(display) != -1) {
    /* This space deliberately left blank */
}
```

`wl_display_dispatch` performs one iteration of the built-in event loop,
and along with other methods like `wl_display_roundtrip` belongs to the
blocking I/O API of libwayland-client. If Wayland events are the only sort
which your program expects, this API is fine and you may skip the rest of
this section. More sophisticated applications need to use the non-blocking
I/O API, where `wl_display_dispatch` is replaced by a combination of:

- `wl_display_get_fd`: returns the FD to poll for `POLLIN` (its ownership
  remains with libwayland-client).

- `wl_display_prepare_read`: signals a thread's intention to read events from
  the socket, and must be paired one-to-one with a following call to
  `wl_display_read_events` or `wl_display_cancel_read` on the same thread.
  Calling either of these two methods without first calling
  `wl_display_prepare_read`, as well as calling `wl_display_prepare_read` twice
  in the same thread, will likely result in deadlocks.

- `wl_display_read_events`: reads all events available on the socket but does
  not dispatch them. It never blocks, instead it returns successfully once
  the socket read yields `EAGAIN`.

- `wl_display_dispatch_pending`: this function doesn't do any I/O, it just
  dispatches pending events if any. To ensure that the event loop isn't put
  to sleep while there are events pending, `wl_display_prepare_read` will fail
  with `EAGAIN` if the queue is not empty. In that case you should call
  `wl_display_dispatch_pending` and try again.

- `wl_display_flush`: attempts to flush any pending requests to the socket,
  returning successfully if all events were written or failing with `EAGAIN`
  if some remain, in which case you should call it again once the FD reports
  `POLLOUT`.

Usage of the non-blocking I/O API can be tricky, but it boils down to:

- Before the event loop is put to sleep (with e.g. `poll`) the very last
  tasks should be:
  - A successful call to `wl_display_prepare_read` (this ensures no other
    thread will read from the socket before our sleep, which would result
    in some events left undispatched before a potentially infinite sleep).
  - A successful (or `EAGAIN`) call to `wl_display_flush` (this ensures
    no requests are left unsent before a potentially indefinite sleep).
- Once the event loop gets out of sleep, the very first task should be
  a call to `wl_display_read_events` (or `wl_display_cancel_read` if the
  event loop is exiting).

This is how an iteration of your event loop would look like, excluding error
checking:

```c
while (wl_display_prepare_read(display) != 0)
    wl_display_dispatch_pending(display);

ret = wl_display_flush(display);
if (ret >= 0)
    fds[WAYLAND_FD_IDX].events &= ~POLLOUT;
else if (ret < 0 && errno == EAGAIN)
    fds[WAYLAND_FD_IDX].events |= POLLOUT;

ret = poll(fds, nfds, -1);
if (has_error(ret)) {
    wl_display_cancel_read(display);
    break;
}
wl_display_read_events(display);

// act on poll results...
```

You may find it odd that integrating libwayland-client requires putting these
tasks strictly around the poll call, or that `wl_display_prepare_read` and
`wl_display_read_events` are performed unconditionally rather than only in
iterations where poll signals `POLLIN` for the FD. This is for two reasons;
first, other threads in your application (possibly from your dependencies, like
a GUI toolkit) might be doing I/O on your display. But even if you only have
one thread, if *any* work of your event loop happens to block on `wl_display`
I/O (either because it calls the blocking I/O API directly or something more
subtle, like locking a mutex held by a thread doing that) that would result in
a deadlock if you've started a `wl_display_prepare_read` but not ended it yet.

As for `wl_display_flush` and `wl_display_dispatch_pending`, calling them right
before polling ensures all events and requests in the queue have been handled
when the thread is put to sleep. This is a minimum, but specialized event loops
may wish to call them at other points too.

So in summary, it *is* possible to relax some of these integration requirements
but that demands extreme care in how the `wl_display` is used by you *and* your
dependencies; as an example, the Mesa EGL integration described later in
this book currently uses blocking I/O under the hood when invoked e.g.
through `eglSwapBuffers`. Mesa will be careful to create its own queue and only
dispatch that queue, so that only Wayland events destined to Mesa are
dispatched while execution is in one of its functions. But it still needs to
read events from the `wl_display`, so when the function returns there might be
new events pending to be dispatched by calling `wl_display_dispatch_pending`.

## Almost there!

At this point you have all of the context you need to set up a Wayland
display and process events and requests. The only remaining step is to allocate
objects to chat about with the other side of your connection. For this, we use
the registry. At the end of the next chapter, we will have our first useful
Wayland client!
