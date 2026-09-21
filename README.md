# Tiny Coroutines (Learning Project)

A small set of C++20 coroutine exercises — a lazy generator and a hand-built custom awaitable with manual suspend/resume — built to understand coroutine mechanics from first principles rather than by using pre-built async libraries.

This is a **basic/skeletal version**, built specifically to internalize the suspend/resume model and the promise-type/awaitable machinery underneath `co_await` and `co_yield` — not a production task scheduler.

## What it does

- A `co_yield`-based generator producing a sequence of values lazily, one at a time, only computing the next value when the caller actually asks for it.
- A hand-written custom awaitable type (`await_ready` / `await_suspend` / `await_resume`) used with `co_await`, first built as an always-ready pass-through, then rebuilt to genuinely suspend.
- A minimal coroutine return type (`Task`) with a nested `promise_type`, satisfying the compiler's required contract for a function to legally be a coroutine.
- Manual suspend/resume: a coroutine that genuinely pauses mid-execution at a `co_await`, hands control back to its caller, and only continues later when the caller explicitly calls `.resume()` on a stored `std::coroutine_handle<>`.

## Concepts used

- The coroutine promise-type contract: `get_return_object()`, `initial_suspend()`, `final_suspend()`, `return_void()`/`return_value()`, `unhandled_exception()` — the machinery that manages a coroutine's lifecycle from the inside.
- The awaitable contract: `await_ready()` (is a value already available), `await_suspend()` (what to do with the now-paused coroutine's handle), `await_resume()` (what value the `co_await` expression produces).
- The two-timeline mental model — a coroutine's own internal execution (paused/running, tracked via its frame and promise type) is a genuinely separate thing from the caller-facing handle (`Task`) representing it from the outside. Most of the boilerplate's confusion resolved once these two roles were kept explicitly separate rather than conflated.
- Why suspension is cooperative and explicit, not automatic: a coroutine that suspends and is never explicitly resumed (via a stored `coroutine_handle`) stays paused forever — nothing wakes it up on its own.
- Why `main()` cannot itself be a coroutine, and why a coroutine's return type cannot be an ordinary type like `int` — both need machinery (a promise type) that plain types and `main`'s special role don't provide.
- The motivating problem coroutines solve: `condition_variable`-based waiting blocks an entire OS thread even for a cheap, short wait. Coroutine suspension pauses only the function's own state (a small heap frame), with zero OS thread blocked — many logical waits can happen concurrently on a single thread.

## What this builds on

Follows directly from the concurrency module — `condition_variable`'s cost (blocking a whole thread to wait) was the concrete motivating problem that made "a function that can pause without blocking a thread" feel like an obvious, necessary tool rather than an arbitrary new feature.

## Key takeaways

- The genuinely hard part of coroutines wasn't any single mechanism — `await_ready`/`await_suspend`/`await_resume` and `promise_type` are each simple in isolation. The difficulty was holding two separate, simultaneously-existing timelines in mind at once (the caller's world, and the coroutine's own paused/running internal state) and correctly sorting which piece of boilerplate served which role.
- Suspension already happens before `await_suspend()` runs — that function doesn't *trigger* pausing, it decides what to do with a coroutine that is already paused. Getting this ordering wrong was an early, genuine point of confusion.
- A coroutine's declared return type and the type that a specific `co_await` expression evaluates to are two completely different things, decided by two different pieces of machinery (`promise_type::get_return_object()` for the former, `await_resume()` for the latter) — conflating them was another real early confusion, resolved once each was traced through an explicit, ordered walkthrough of a single `co_await` call.
- This is base-shape understanding, not mastery — genuinely internalizing coroutines further (a real multi-coroutine scheduler, actual async I/O) needs a project that requires it, not more toy exercises in isolation.

## Status

Core mechanics working and understood: a working lazy generator, a working custom awaitable demonstrating both the always-ready and genuinely-suspending paths, and manual handle-based resume proven to work end to end. Not yet extended into an actual multi-coroutine scheduler (deciding which of several paused coroutines to resume next) or real-world use (async I/O, task queues) — flagged as a deliberate next step for when a real project calls for it.
