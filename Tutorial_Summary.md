# [Tokio Tutorial](https://tokio.rs/tokio/tutorial) Summary

## Hello Tokio

Running `async fn` doesn't execute immediately, but returns a value representing the operation. To run the operation, `await` should be used.

`#[tokio::main]` is macro for initializing runtime and running the given function.

## Spawning

Task is the unit of execution managed by the scheduler. Each only takes 64 bytes, safe to spawn a ton. Create a task by passing a async block to `tokio::spawn`.

Task's lifetime must be `'static`, meaning any data in task must be `'static`, spawned task must not contain any reference to data outside task (know the [difference between `'static` and `&'static`](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program)). To share data between tasks, use `Arc`.

When a task is suspended, it can be moved and ran on another thread, thereby a task must be `Send`, meaning to be able to move tasks between threads all data in task must be `Send`.

## Shared State

### Sharing states on Tokio

- Mutex + Arc. Used for simple data. Explained more in the following section.
- State management task, spawn a task for managing state, uses messaging to share state. Used for data that requires asynchronous work (e.g., I/O primitives).

### Arc

Arc, short for Atomically Reference Counted, shares access to the value with its clones while being thread-safe, distinct from Rc (Reference Counted, not thread-safe). When cloned, only the reference is copied, not its value. It counts the number of clones, dropping the stored value when the last clone is destroyed.

handle: a value that provides access to some shared state (e.g., `Arc` instance)

### Mutex

Mutex, short for Mutual Exclusion, uses lock to prevent two threads from accessing data at the same time (data race).
`Mutex.lock` returns a `MutexGuard`. Only one instance of `MutexGuard` can exist and if another thread already owns a `MutexGuard` instance, it will sleep until it goes out of scope.

Compiler prevents awaiting with `MutexGuard` in scope, because calling `lock` puts the thread to sleep if it already exists (deadlock). Condition for error is

1. Current task can be moved across thread.
1. Mutex that doesn't implement `Send` for `MutexGuard`.

If one of the conditions is not satisfied, error doesn't occur. Solutions are,

1. Make `MutexGuard` go out of scope before `await` (code block, wrapper struct)
1. Use `tokio::sync::Mutex`.
1. Single threaded with `tokio::task::spawn_local`

Note, using `drop` before `await` doesn't work as rust compiler calculates `Send` based on the scope information only.
Other workarounds, `MutexGuard` implementations that is `Sync`, spawning task that doesn't require `Sync` (e.g., `tokio:task::LocalSet`), are not recommended as it could result in deadlock.

```rust
// This won't work. Rust compiler only sees scope.
{
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock); // lock goes out of scope here

    do_something().await;
}

// Solutions for MutexGuard not Sync
// 1-1.Make MutexGuard go out of scope before await
{
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // lock goes out of scope here

    do_something().await;
}

// 1-2.Wrapper struct
struct CanIncrement {
    mutex: Mutex<i32>,
}
impl CanIncrement {
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

// 2.tokio::sync::Mutex. Tokio Mutex comes with a higher cost
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
}
// Using one of dedicated struct or tokio Mutex is recommended.
```

#### Wrapper Struct

Creating a wrapper struct instead of passing `Arc<Mutex>` is recommended for following reasons.

- Hide implementation detail, makes code refactoring easier.
- With raw `Arc`, type signatures are clutter.
- Calling `lock` every time is a clutter.
- Wrapper prevent holding `MutexGuard` for too long.

##### with\_\* pattern

Returning a `MutexGuard` should be avoided.
When `MutexGuard` needs to be passed as a function parameter, `with_*` pattern can be used, instead of creating a new method.

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
pub struct SharedMap { // Wrapper Struct
    inner: Arc<Mutex<SharedMapInner>>,
}

struct SharedMapInner {
    data: HashMap<i32, String>,
}

impl SharedMap {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(SharedMapInner {
                data: HashMap::new(),
            }))
        }
    }

    // MutexGuard lives only inside this method
    // providing clean type signatures and easy refactoring
    pub fn insert(&self, key: i32, value: String) {
        let mut lock = self.inner.lock().unwrap();
        lock.data.insert(key, value);
    }

    // with_* pattern
    pub fn with_value<F, T>(&self, key: i32, func: F) -> T
    where
        F: FnOnce(Option<&str>) -> T,
    {
        let lock = self.inner.lock().unwrap();
        func(lock.data.get(&key).map(|string| string.as_str()))
    }
}
```

### Managing contention

Large number of threads requiring mutex will lead to contention, blocking all tasks waiting to be executed. Switching to `tokio::sync::Mutex` from `std::sync::Mutex` is not a solution for solving contention problem.

#### Solutions

- Dedicated state management task, use message passing.
- Sharding (ConcurrentHashMap)
- Avoid using mutex

#### Sharding (ConcurrentHashMap) crates

- [dashmap](https://docs.rs/dashmap/latest/dashmap/)
- [leapfrog](https://docs.rs/leapfrog/latest/leapfrog/)
- [flurry](https://docs.rs/flurry/latest/flurry/#)

### References

- [Shared State (Tokio Tutorial)](https://tokio.rs/tokio/tutorial/shared-state)
- [Shared mutable state in Rust (Blog)](https://draft.ryhl.io/blog/shared-mutable-state/)
- [[Java] ConcurrentHashMap은 어떻게 Thread-safe한가?](https://velog.io/@alsgus92/ConcurrentHashMap%EC%9D%98-Thread-safe-%EC%9B%90%EB%A6%AC)
- [[Java] ConcurrentHashMap이란 무엇일까?](https://devlog-wjdrbs96.tistory.com/269)

## Channels, sharing client with multiple tasks

To share a client with multiple tasks without locking it, we use client task with channels.
Several Tokio's channel primitives exist which differ in the number of producer and consumer.
Basic workflow is as follows,

1. Define message type (with capacity)
1. Create and pass channel to the task
1. `await` on the responder to receive message

### References

- [Channels (Tokio Tutorial)](https://tokio.rs/tokio/tutorial/channels)

## I/O
