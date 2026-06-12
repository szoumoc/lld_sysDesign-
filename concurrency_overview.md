# Concurrency in Low-Level Design (LLD) Interviews

Concurrency problems in LLD interviews almost always reduce to one of three categories:

1. **Correctness** — shared state gets corrupted when multiple threads touch it at once.
2. **Coordination** — threads need to hand work off to each other or wait for one another.
3. **Scarcity** — a resource is limited and many threads are competing for it.

This note works through each category with the failure mode, the fix, and code in Go, C++, Python, and Java.

---

## 1. Correctness

Correctness problems happen when multiple threads read and write the **same shared state**, and the interleaving of their operations produces a result that wouldn't happen if everything ran one-at-a-time (serially).

There are two flavors of this that show up constantly in interviews.

### 1.1 Check-Then-Act

This is the classic race condition shape:

```
if (condition is true) {   // CHECK
    do something           // ACT
}
```

The problem: between the "check" and the "act", another thread can sneak in and invalidate the condition. Two threads both check `seatAvailable == true`, both see `true`, both proceed to book the seat. Now you've oversold by one.

**Fix:** wrap the check *and* the act in a **lock (mutex)** so the whole sequence becomes one atomic unit — no other thread can interleave in the middle.

The critical rule: the lock must cover **both** the check and the act. Locking only the "act" part doesn't help, because another thread can still pass the check before you acquire the lock.

#### Go

```go
type SeatBooking struct {
    mu        sync.Mutex
    available bool
}

func (s *SeatBooking) BookSeat() bool {
    s.mu.Lock()
    defer s.mu.Unlock()

    if !s.available {       // CHECK
        return false
    }
    s.available = false     // ACT
    return true
}
```

#### C++

```cpp
class SeatBooking {
    std::mutex mu;
    bool available = true;

public:
    bool bookSeat() {
        std::lock_guard<std::mutex> lock(mu);

        if (!available) {    // CHECK
            return false;
        }
        available = false;   // ACT
        return true;
    }
};
```

#### Python

```python
import threading

class SeatBooking:
    def __init__(self):
        self.lock = threading.Lock()
        self.available = True

    def book_seat(self):
        with self.lock:
            if not self.available:   # CHECK
                return False
            self.available = False   # ACT
            return True
```

#### Java

```java
public class SeatBooking {
    private final Object lock = new Object();
    private boolean available = true;

    public boolean bookSeat() {
        synchronized (lock) {
            if (!available) {     // CHECK
                return false;
            }
            available = false;    // ACT
            return true;
        }
    }
}
```

**Lock granularity options:**

- **Coarse-grained**: one lock for the entire object/system. Simple, but every booking serializes against every other booking, even for unrelated seats.
- **Fine-grained**: one lock per resource (e.g., per seat, per locker, per inventory item). Lets unrelated operations proceed in parallel, but more locks = more chances to deadlock if you're not careful about ordering.
- **Read-write lock**: if reads vastly outnumber writes (e.g., checking seat availability happens far more than booking), a read-write lock lets many readers proceed concurrently while writers get exclusive access.

In interviews, start coarse-grained, then offer fine-grained locking as an optimization once the interviewer pushes on throughput.

---

### 1.2 Read-Modify-Write

This is the simpler cousin of check-then-act, but it bites just as often. A single shared variable (a counter, a flag, a balance) is read, modified, and written back:

```
value = value + 1
```

This looks like one operation but is actually three machine steps: read `value` into a register, increment the register, write it back. If two threads interleave these steps, one increment can be lost entirely.

**Fix:** for a **single variable**, use an **atomic** operation. Atomics use CPU-level instructions (like compare-and-swap) to perform read-modify-write as one uninterruptible step — no lock needed.

The key constraint: atomics only help when you're updating **one variable**. The moment your operation needs to update two related fields together (e.g., "decrement `availableCount` AND append to `bookedList`"), you're back to check-then-act territory and need a lock.

#### Go

```go
import "sync/atomic"

var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}

func read() int64 {
    return atomic.LoadInt64(&counter)
}
```

#### C++

```cpp
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    counter.fetch_add(1, std::memory_order_relaxed);
}

int read() {
    return counter.load(std::memory_order_relaxed);
}
```

#### Python

Python has no native atomic integer type. The GIL prevents some classes of corruption for simple bytecode operations, but compound operations like `+=` are **not** guaranteed atomic across all implementations, so the idiomatic and safe approach is a lock:

```python
import threading

class AtomicCounter:
    def __init__(self):
        self._lock = threading.Lock()
        self._value = 0

    def increment(self):
        with self._lock:
            self._value += 1

    def get(self):
        with self._lock:
            return self._value
```

#### Java

```java
import java.util.concurrent.atomic.AtomicLong;

public class Counter {
    private final AtomicLong counter = new AtomicLong(0);

    public void increment() {
        counter.incrementAndGet();
    }

    public long get() {
        return counter.get();
    }
}
```

**When to reach for what:**

| Situation | Tool |
|---|---|
| Single counter / flag / id generator | Atomic |
| Multiple fields must change together | Lock |
| "Check this, then act on it" | Lock (covering both check and act) |

---

## 2. Coordination

Coordination problems happen when one thread **produces** work and another thread **consumes** it — and the two need to hand off cleanly without wasting CPU or adding unnecessary latency.

### The motivating scenario

Imagine a user signs up, and as part of that flow you need to send a welcome email. You don't want the user-facing request to block on an email send (which might take 500ms+ due to a slow SMTP provider). So you push the "send email" task onto a queue, and a separate **worker thread** picks it up and executes it asynchronously.

This introduces two new questions:

1. **How does the consumer know work has arrived?** It can't just sit there doing nothing — but it also can't constantly ask "is there work yet? is there work yet?"
2. **What happens if producers add work faster than consumers can process it?** The queue grows unbounded and you eventually run out of memory.

### 2.1 The Naive (Bad) Approach: Busy-Wait Loop

```python
# DON'T DO THIS
while True:
    if not queue.empty():
        task = queue.get()
        process(task)
```

This is a **busy-wait** or **spin loop**. The consumer thread runs constantly, checking the queue over and over, even when there's nothing to do. On a single core this starves other threads of CPU time; on any machine it burns 100% of a core for no reason. This is almost never the right answer in an interview.

### 2.2 Better but Flawed: Sleep-and-Poll

```python
import time

while True:
    if not queue.empty():
        task = queue.get()
        process(task)
    else:
        time.sleep(0.5)  # poll every 500ms
```

This fixes the CPU-burning problem — the thread sleeps instead of spinning. But it introduces a new problem: **latency**. If a task arrives right after the thread goes to sleep, it sits there for up to 500ms before the next poll picks it up. Now your "send email" task, which should execute near-instantly once it's in the queue, has up to 500ms of artificial delay tacked on — every single time. Shrinking the sleep interval reduces latency but brings back the CPU-burning problem. You can't win with polling.

### 2.3 The Correct Approach: Blocking Queue

A **blocking queue** is built on top of a lock and a **condition variable**. It gives you:

- `put(item)` — adds an item; if the queue is **full**, the calling (producer) thread blocks until space frees up.
- `take()` — removes an item; if the queue is **empty**, the calling (consumer) thread blocks until an item arrives.

Crucially, "blocks" here doesn't mean spinning or sleeping on a timer — the OS puts the thread to sleep with zero CPU usage, and the thread is woken up **immediately** the moment an item is available (because `put()` signals the condition variable). This gives you the best of both worlds: zero wasted CPU, and near-zero latency.

This also elegantly solves the "producers faster than consumers" problem: once the queue is full, `put()` blocks the producer, which naturally applies **backpressure** — the producer slows down to match the consumer's pace instead of growing the queue without bound.

#### Go

Go channels are the idiomatic blocking queue — a buffered channel gives you bounded capacity with built-in blocking semantics.

```go
package main

type EmailTask struct {
    To      string
    Subject string
    Body    string
}

func main() {
    taskQueue := make(chan EmailTask, 100) // bounded capacity = 100

    // Producer
    go func() {
        taskQueue <- EmailTask{To: "user@example.com", Subject: "Welcome!"}
        // blocks here if queue already has 100 items
    }()

    // Consumer (worker)
    go func() {
        for task := range taskQueue {
            sendEmail(task) // blocks here if queue is empty
        }
    }()
}
```

#### C++

C++ has no built-in blocking queue, so it's composed manually from `std::queue`, `std::mutex`, and `std::condition_variable`.

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>

template <typename T>
class BlockingQueue {
    std::queue<T> queue;
    std::mutex mu;
    std::condition_variable notEmpty;
    std::condition_variable notFull;
    size_t capacity;

public:
    explicit BlockingQueue(size_t capacity) : capacity(capacity) {}

    void put(T item) {
        std::unique_lock<std::mutex> lock(mu);
        notFull.wait(lock, [this] { return queue.size() < capacity; });
        queue.push(std::move(item));
        notEmpty.notify_one();
    }

    T take() {
        std::unique_lock<std::mutex> lock(mu);
        notEmpty.wait(lock, [this] { return !queue.empty(); });
        T item = std::move(queue.front());
        queue.pop();
        notFull.notify_one();
        return item;
    }
};
```

#### Python

Python's `queue.Queue` is a fully built-in blocking queue.

```python
import queue
import threading

task_queue = queue.Queue(maxsize=100)  # bounded

def producer():
    task = {"to": "user@example.com", "subject": "Welcome!"}
    task_queue.put(task)  # blocks if queue has 100 items

def worker():
    while True:
        task = task_queue.get()  # blocks if queue is empty
        send_email(task)
        task_queue.task_done()

threading.Thread(target=worker, daemon=True).start()
```

#### Java

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class EmailService {
    private final BlockingQueue<EmailTask> taskQueue = new LinkedBlockingQueue<>(100);

    // Producer
    public void enqueueEmail(EmailTask task) throws InterruptedException {
        taskQueue.put(task); // blocks if queue is full
    }

    // Consumer (worker thread)
    public void worker() {
        while (true) {
            try {
                EmailTask task = taskQueue.take(); // blocks if queue is empty
                sendEmail(task);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```

**Summary of the progression:**

| Approach | CPU usage when idle | Latency when work arrives |
|---|---|---|
| Busy-wait loop | 100% (wasteful) | Near-zero |
| Sleep-and-poll | Low | Up to the sleep interval |
| Blocking queue (condition variable) | Zero | Near-zero |

---

## 3. Scarcity

Scarcity problems happen when you have a **limited resource** — a fixed number of database connections, a third-party API that only allows N concurrent requests, a fixed-size thread pool — and more threads want to use it than there are units available.

### 3.1 Semaphores

A **semaphore** is a counting lock. Instead of "locked / unlocked", it holds a count of **permits**. A thread calls `acquire()` to take a permit (blocking if none are available) and `release()` to give it back.

**Example:** you're calling a third-party API that allows at most 10 concurrent requests. Create a semaphore with 10 permits. Every caller must acquire a permit before making the request and release it afterward.

```python
import threading

api_semaphore = threading.Semaphore(10)

def call_external_api(payload):
    api_semaphore.acquire()
    try:
        return external_api.send(payload)
    finally:
        api_semaphore.release()
```

#### Why `try / finally` (or its equivalent) is mandatory

If `external_api.send(payload)` throws an exception, and you don't release the permit, that permit is **leaked forever**. The semaphore's count effectively shrinks by one, permanently, every time an exception happens on that code path. Do this 10 times and your "10 concurrent requests" pool is down to zero — and every future caller blocks forever, even though the resource isn't actually in use.

This is **exception bubbling**: the exception propagates up out of the critical section, skipping any cleanup code that comes after the risky call — unless that cleanup is in a `finally` block (or the language's equivalent: `defer` in Go, RAII destructors in C++, `with` in Python, `try-with-resources`/`finally` in Java).

The pattern is always: **acquire outside the try, do the risky work inside the try, release in finally.** Acquiring inside the `try` is a subtle bug — if `acquire()` itself throws (rare, but possible with timeouts), you'd attempt to release a permit you never actually acquired.

#### Go

```go
import "golang.org/x/sync/semaphore"

var sem = semaphore.NewWeighted(10)

func callExternalAPI(ctx context.Context, payload Payload) (Result, error) {
    if err := sem.Acquire(ctx, 1); err != nil {
        return Result{}, err
    }
    defer sem.Release(1) // guaranteed to run even on panic/early return

    return externalAPI.Send(payload)
}
```

#### C++

```cpp
#include <semaphore>

std::counting_semaphore<10> apiSemaphore{10};

Result callExternalApi(const Payload& payload) {
    apiSemaphore.acquire();
    try {
        return externalApi.send(payload);
    } catch (...) {
        apiSemaphore.release();
        throw; // re-throw after cleanup
    }
    apiSemaphore.release();
    return result;
}

// Cleaner: use an RAII guard so release() can't be forgotten
struct SemaphoreGuard {
    std::counting_semaphore<10>& sem;
    explicit SemaphoreGuard(std::counting_semaphore<10>& s) : sem(s) { sem.acquire(); }
    ~SemaphoreGuard() { sem.release(); }
};

Result callExternalApiRAII(const Payload& payload) {
    SemaphoreGuard guard(apiSemaphore);
    return externalApi.send(payload); // released automatically, even on exception
}
```

#### Java

```java
import java.util.concurrent.Semaphore;

public class ExternalApiClient {
    private final Semaphore apiSemaphore = new Semaphore(10);

    public Result callExternalApi(Payload payload) throws InterruptedException {
        apiSemaphore.acquire();
        try {
            return externalApi.send(payload);
        } finally {
            apiSemaphore.release(); // always runs, even if send() throws
        }
    }
}
```

---

### 3.2 Connection Pools

A **connection pool** is the scarcity pattern applied to expensive-to-create resources, like database connections. Creating a new connection is costly (TCP handshake, auth, etc.), so instead of creating one per request, you pre-create a fixed number of connections at startup and let threads **borrow** and **return** them.

A connection pool is really a **blocking queue of pre-created resources**, combined with the same acquire/release discipline as a semaphore:

- `getConnection()` → `take()` from the queue (blocks if empty, i.e., all connections currently in use)
- `releaseConnection(conn)` → `put()` back into the queue

#### Python

```python
import queue

class ConnectionPool:
    def __init__(self, size=10):
        self._pool = queue.Queue(maxsize=size)
        for _ in range(size):
            self._pool.put(create_new_connection())

    def get_connection(self):
        return self._pool.get()  # blocks if none available

    def release_connection(self, conn):
        self._pool.put(conn)

# Usage
pool = ConnectionPool(size=10)

def run_query(sql):
    conn = pool.get_connection()
    try:
        return conn.execute(sql)
    finally:
        pool.release_connection(conn)  # always returned, even on error
```

#### Java

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class ConnectionPool {
    private final BlockingQueue<Connection> pool;

    public ConnectionPool(int size) {
        pool = new LinkedBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            pool.add(createNewConnection());
        }
    }

    public Connection getConnection() throws InterruptedException {
        return pool.take(); // blocks if none available
    }

    public void releaseConnection(Connection conn) {
        pool.offer(conn);
    }

    public ResultSet runQuery(String sql) throws Exception {
        Connection conn = getConnection();
        try {
            return conn.execute(sql);
        } finally {
            releaseConnection(conn); // always returned, even on exception
        }
    }
}
```

#### Go

```go
type ConnectionPool struct {
    pool chan *Connection
}

func NewConnectionPool(size int) *ConnectionPool {
    p := &ConnectionPool{pool: make(chan *Connection, size)}
    for i := 0; i < size; i++ {
        p.pool <- createNewConnection()
    }
    return p
}

func (p *ConnectionPool) GetConnection() *Connection {
    return <-p.pool // blocks if none available
}

func (p *ConnectionPool) ReleaseConnection(c *Connection) {
    p.pool <- c
}

func RunQuery(p *ConnectionPool, sql string) (Result, error) {
    conn := p.GetConnection()
    defer p.ReleaseConnection(conn) // always runs

    return conn.Execute(sql)
}
```

#### C++

```cpp
class ConnectionPool {
    BlockingQueue<Connection*> pool; // from section 2.3

public:
    explicit ConnectionPool(size_t size) : pool(size) {
        for (size_t i = 0; i < size; ++i) {
            pool.put(createNewConnection());
        }
    }

    Connection* getConnection() {
        return pool.take(); // blocks if none available
    }

    void releaseConnection(Connection* conn) {
        pool.put(conn);
    }
};

// RAII guard ensures the connection is always returned
class ConnectionGuard {
    ConnectionPool& pool;
    Connection* conn;
public:
    explicit ConnectionGuard(ConnectionPool& p) : pool(p), conn(p.getConnection()) {}
    ~ConnectionGuard() { pool.releaseConnection(conn); }
    Connection* get() { return conn; }
};

Result runQuery(ConnectionPool& pool, const std::string& sql) {
    ConnectionGuard guard(pool);
    return guard.get()->execute(sql); // connection returned automatically
}
```

---

## Putting It All Together

| Category | Symptom | Primitive | Key Insight |
|---|---|---|---|
| Correctness — check-then-act | Two threads pass a check before either acts | Lock / Mutex | Lock must cover the **entire** check-and-act sequence |
| Correctness — read-modify-write | Lost updates to a single variable | Atomic | Only works for **one** variable; multi-field updates need a lock |
| Coordination | Producer/consumer handoff; busy loops or laggy polling | Blocking Queue (condition variable) | Blocking gives zero idle CPU **and** near-zero latency |
| Scarcity | More demand than available resource units | Semaphore / Connection Pool | Acquire outside `try`, release in `finally` — never let exceptions leak permits |

### Interview Strategy

1. **Identify the shared state first.** What variable(s), object(s), or resource(s) are multiple threads touching?
2. **Classify the problem.** Is it correctness (state corruption), coordination (handoff/ordering), or scarcity (limited resource)? Most problems start as correctness and grow into the other two as follow-ups.
3. **Pick the smallest tool that fixes it.** Atomic before lock. Lock before redesigning data structures. Blocking queue before custom condition-variable logic.
4. **Always pair acquire with guaranteed release.** `defer` / RAII / `finally` / `with` — pick whichever your language gives you, but never skip it.
5. **State your assumptions about scale.** A coarse-grained lock is a perfectly good starting answer; only move to fine-grained locking, lock-free structures, or sharding if the interviewer pushes on throughput.