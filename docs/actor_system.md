# Purpose

The library needs a way to run stateful, message-driven concurrent tasks. Traditional async tasks don't have built-in message passing or lifecycle hooks.
Actors solve this by providing isolated mutable state, sequential message processing (no data races), and explicit lifecycle hooks(`on_start`, `on_stop`).
The spawning flow must also ensure actors are region-owned. They cannot outlive their region - and integrate with runtime's capability context system for cancellation, budgeting, and observability.

# Actor spawning flow

When you spawn an actor via `Scope::spawn_actor`, the runtime first create a **bounded mpsc channel** to serve as the actor's mailbox.
This ensures backpressure when messages arrive faster than the acctor can process them.

A separate oneshot channel is created to return the actor's final state when you call `join()`.

The runtime then creates a task record in the global state, assigning a unique `TaskId` that become the actor's identity.
TODO: We need to rethink about the global state. We don't want the global state, the state is provided so it can manage in structure.

A **child capability context** is constructed with the region's budget, observability, and I/O driver handles. This gives the actor access to runtime capabilities without ambien authority.

Finally, the actor loop future is wrapped in `CatchUnwind` (for panic recovery) and stored as a `StoredTask` that the executor can poll. The returned `ActorHandle` contains the sender for the mailbox, allowing you to send messages to the actor from anywhere in your code.

```rust
pub fn spawn_actor<A: Actor>(&self)
```

The `spawn_actor` do the following things:

- Create MPSC mailbox
- Create result channel
- Create task record
- Create child capability context
- Wrap in StoredTask

# Message Processing

Actors are stateful, message-driven concurrent units that must process messages sequentially without data races.

The core problems is: how do we ensure **deterministic shutdown** where no committed message is silently dropped, even when actors are cancelled mid-execution?
Traditional async systems lose messages on cancellation, violating invariants. 

The actor solves this with a **four-phase lifecycle** that guarantees every successfully-sent message is handled before cleanup runs.

The actor loop executes four distinct phases:

- Initialization calls `on_start` for setup before any message processing. Use this for initialization requiring the capability context.
- Message loop receives from the bounded MPSC mailbox and calls `handle` sequentially.  The actor has exclusive access to its state during handling - no concurrent reads or writes. The loop checks cancellation before each receive and breaks on disconnect or cancel signals.
- Mailbox drain is the critical shutdown invariant. After the loop exists, the actor drains remaining bufferred messages. This ensures the **two-phase mailbox guarantee**: no message that was successfully committed is silently dropped. Every committed message is processed before `on_stop` runs.
- Cleanup call `on_stop` after the drain phase. Use this for resource cleanup, flushing state, or logging final metrics.

The state transitions atomically through `ActorStateCell` (Created -> Running -> Stopping -> Stopped), allowing external observers to track lifecycle without lcoks. The entire loop is wrapped in `CatchUnwind` at spawn time, converting panics into `JoinError::Panicked` for supervision handling.

# Supervised restart

Long-running systems face transient failures - network hiccups temporary resource exhaustion or bugs that only trigger under specific conditions. Rather than letting these failures propagate and crash the entire system, the actor pattern provides automatic restart with configurable policies.

Actor flow wraps the standard actor loop in a restart -aware layer. When an actor runs, it's wrapped in `CatchUnwind` to detect panics. If the actor completes normally, the loop returns immediately.

If it panics, the supervisor consults its strategy to decide between Restart, Stop, or Escalate.

For restarts, a **factor closure** creates fresh actor instances. The mailbox persists across restarts, so messages sent during the restart window are bufferred and processed by the new instance. The supervisor applies rate limitting (max restarts per time window) and backoff strategies to prevent restart atoms and give transient failures time to resolve.

# Actor State Transitions

Actors need to communicate their lifecycle status to external observers (like ActorRef.is_alive()) without locking. The problem is that multiple threads may query an actor's state while it's transitioning between phass. 

A naive approach using a mutex would cause contention on hot paths. The solution is **atomic state encoding** - mapping the four licycle states to u8 values that can be read/written with lock-free atomic operations.

The actor lifecycle has four states: **Created**, **Running**, **Stopping**, and **Stopped**. These are encoded as 0, 1, 2 and 3 respectively in the `encode()` function. The `ActorStateCell` wraps an `AtomicU8` and provides `load()` and `store()` methods with Acquire/Release ordering for thread safety.

Unknown values (4+) fail-safe to Stopped, ensureing robustness again corruption.

# Message sending

In traditional async systems, sending a message to an actor can fail silently if cancelled mid-operation. If cancellation occurs after a message is queued but before delivery completes, the message may be lost without any indication to the sender. This is unacceptable for systems where message reliability is critical - you need guarantees that every message you successfully "send" is eiter delivered or explicitly rejected, with no silent drops.

We solves this with a **two-phase reserve/commit pattern** for all message sending. When you call `AntorHandle::send()`, it delegates to the underlying MPSC channel which implements this pattern.

The first phase is `reserve()`, which atomically reserves a slot in actor's bounded mailbox. If the mailbox is full or the actor has stopped, this return an error immediately.

Once you have a permit, the second phase is `permit.send()`, which commits the message and **cannot fail**.

This design ensure **cancel-safety**. If cancellation occurs during the reserve phase, the permit is dropped and no message is lost. If cancellation occurs after reserve but before commit, when sender commit the message, the sender receives `SendError::Disconnedted(value))`. The actor's mailbox drain phase guarantees that every successfully committed message is processed beofre the actor's on_stop` runs, eliminating silent drops entirely.