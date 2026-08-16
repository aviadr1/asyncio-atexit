# asyncio atexit

Adds atexit functionality to asyncio:

```python
import asyncio_atexit

async def close_db():
    await db_connection.close()

asyncio_atexit.register(close_db)
```

[atexit][] is part of the standard library,
and gives you a way to register functions to call when the interpreter exits.

[atexit]: https://docs.python.org/3/library/atexit.html

asyncio doesn't have equivalent functionality to register functions
when the _event loop_ exits:

This package adds functionality that can be considered equivalent to `atexit.register`,
but tied to the event loop lifecycle. It:

1. accepts both coroutines and synchronous functions
1. should be called from a running event loop
1. calls registered cleanup functions when the event loop closes
1. only works if the application running the event loop calls `close()`

---

## About this fork

This is a fork of [minrk/asyncio-atexit](https://github.com/minrk/asyncio-atexit) (MIT).
It keeps the same import name and API, so it is a drop-in replacement, and adds **time bounds**
and an **exit watchdog**.

### Why

Upstream cannot bound a cleanup callback that blocks. A blocked callback means the event loop
never closes and the process never exits — and when a supervisor reclaims resources on process
exit, one wedged callback leaks that resource permanently.

The incident that motivated the fork: a LaunchDarkly client closed from an atexit callback.
Under degraded DNS, `ldclient.close()` → `EventProcessor.stop()` → `FixedThreadPool.wait()` is
a bare `threading.Event.wait()` with no timeout, waiting on flush-worker threads stuck in
`getaddrinfo()` — urllib3's connect/read timeouts don't bound name resolution. The process hung
forever. The job runner supervising it released its concurrency slot only on process exit, so
every occurrence permanently consumed a slot until the whole pool was wedged.

Upstream can't bound that, for two compounding reasons:

1. It runs callbacks as `f = callback()` **on the loop thread**, awaiting only if the result is
   awaitable. A *synchronous* blocking callback pins the loop thread outright.
2. Even wrapping that await in `asyncio.wait_for` wouldn't help — `wait_for`'s timeout is a
   timer callback scheduled on that same loop, and a blocked loop thread can never run it.

**The second point is why this is a fork rather than a patch.** The obvious fix — add a timeout
upstream — bounds only callbacks that yield to the loop, which is exactly the class that did
*not* cause the incident:

| | `loop.close()` with an 8s blocking callback, 1s declared bound |
|---|---|
| upstream `asyncio_atexit` | **8.00s** |
| upstream + caller's `asyncio.wait_for` | **8.01s** |
| this fork | **1.00s** |

### What changed

| | upstream | this fork |
|---|---|---|
| Sync callback that blocks | pins the loop thread forever | runs on a daemon thread, abandoned at its deadline |
| Async callback that never resolves | awaited forever | abandoned at its deadline |
| Callback raises | caught, printed to stderr, continues | same, but off-thread failures are carried back rather than hitting `threading.excepthook` |
| Callback hangs | every later callback is skipped | later callbacks still run |
| Process wedged in shutdown | hangs forever | optional watchdog forces exit |

```python
# Per-callback bound; the default is 10s.
asyncio_atexit.register(close_db, timeout=5)

# Opt in to the backstop. Off by default: it ends in os._exit, so a process that closes a
# loop and then keeps running must not be killed by it.
asyncio_atexit.enable_exit_watchdog(grace_seconds=90)
```

The watchdog is armed **before the first callback runs**, so it covers the callbacks
themselves, the real `loop.close()`, and interpreter finalization. That ordering is why it
lives in the dispatcher rather than being registered as just another callback — as a callback
it would sit behind whatever registered first, and a hang there would mean it never armed.

### Invariants

Each has a test named after it in `test_asyncio_atexit.py`:

| | |
|---|---|
| **I1** | `loop.close()` returns within `sum(timeouts) + epsilon`, whatever the callbacks do |
| **I2** | A callback that overruns or raises never prevents a later callback from running |
| **I3** | No callback exception reaches `threading.excepthook` |
| **I4** | The watchdog is armed before the first callback runs |
| **I5** | The watchdog never arms unless the process explicitly opts in |

Measured by the suite:

| Scenario | Bound | Actual |
|---|---|---|
| Blocking sync callback | 1s | 1.01s |
| Three hung callbacks | Σ = 3s | 3.05s |
| Wedged process, watchdog grace 2s | 2s | 2.12s (would otherwise hang forever) |

### Two bounds, and which one wins

The per-callback timeout and the watchdog grace are independent and can disagree: N callbacks
each allowed the default 10s can sum past a 90s grace.

When they disagree **the watchdog wins, by design**. The per-callback timeout bounds one hook
so the *others still get to run* (I2); the watchdog bounds the process so a supervisor can
reclaim it, and it's a backstop rather than a participant. A process already wedged for the
full grace has nothing left worth waiting for.
