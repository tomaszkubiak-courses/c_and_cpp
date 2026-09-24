# C++ Day 27 — Concurrency: threads, locks, atomics, async

**Goal:** run work in parallel with `std::jthread` and `std::async`, protect shared data with
mutexes and atomics, coordinate threads with condition variables, and find data races with
ThreadSanitizer.

Compile today's programs with `-pthread` on Linux (CMake: `find_package(Threads REQUIRED)` and link
`Threads::Threads`).

## Concepts

### Threads

A **thread** is an independent flow of execution within your program. All threads share the same
memory — which is what makes them useful and dangerous.

```cpp
#include <iostream>
#include <string>
#include <thread>
#include <vector>

int main()
{
    std::vector<std::jthread> workers;
    for (int id = 0; id < 4; ++id) {
        workers.emplace_back([id] {                     // each thread runs this lambda
            std::cout << "worker " + std::to_string(id) + " running\n";   // one << : less interleaving
        });
    }
}   // jthread joins automatically in its destructor: main waits for all four
```

`std::jthread` (C++20) is RAII for threads: its destructor **joins** (waits for the thread to
finish). The older `std::thread` terminates the program if destroyed without `join()` or
`detach()` — prefer `jthread`. The output order varies between runs: threads run concurrently, and
the OS schedules them.

`std::thread::hardware_concurrency()` tells you how many threads the hardware can run at once.

### Data races

When two threads access the same memory, at least one writes, and nothing orders the accesses,
that's a **data race** — undefined behavior:

```cpp
// BUG: data race
#include <iostream>
#include <thread>

int main()
{
    long counter = 0;
    {
        std::jthread a{[&] { for (int i = 0; i < 1'000'000; ++i) ++counter; }};
        std::jthread b{[&] { for (int i = 0; i < 1'000'000; ++i) ++counter; }};
    }
    std::cout << counter << '\n';   // rarely 2000000
}
```

`++counter` is really *read, add, write*; two threads interleave those steps and lose updates.
Build with `-fsanitize=thread -g` and ThreadSanitizer reports the race with both stack traces.

### Mutexes and locks

A `std::mutex` lets only one thread at a time into a **critical section**. Lock it with an RAII
guard, never with manual `lock()`/`unlock()` (an exception or early `return` would leave it locked):

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int main()
{
    long counter = 0;
    std::mutex m;
    auto work = [&] {
        for (int i = 0; i < 1'000'000; ++i) {
            std::scoped_lock lock{m};        // locks here, unlocks at the end of the block
            ++counter;
        }
    };
    {
        std::jthread a{work}, b{work};
    }
    std::cout << counter << '\n';            // 2000000, always
}
```

- `std::scoped_lock` locks one or **several** mutexes without deadlock (it orders the locking).
- `std::unique_lock` is a movable lock you can unlock early — needed for condition variables.
- **Deadlock:** thread 1 holds A and waits for B while thread 2 holds B and waits for A. Avoid it by
  always locking in the same order, or locking both at once with `std::scoped_lock{a, b}`.
- Keep critical sections short; never call unknown code (callbacks) while holding a lock.

### Atomics

For a single variable, `std::atomic<T>` makes operations indivisible without a mutex:

```cpp
#include <atomic>
std::atomic<long> counter{0};
++counter;                      // atomic read-modify-write
counter.fetch_add(5);
```

Much faster than a mutex for counters and flags. Only individual operations are atomic, though:
`if (x == 0) x = 1;` is still two operations. The memory-ordering options (`memory_order_relaxed`,
`acquire`, `release`) are an advanced topic — the default (`seq_cst`) is correct.

### Condition variables: waiting for something to happen

A classic **producer–consumer queue**: producers push work, consumers wait until there's work.

```cpp
#include <atomic>
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <optional>
#include <queue>
#include <thread>
#include <utility>
#include <vector>

template <typename T>
class WorkQueue {
public:
    void push(T item)
    {
        {
            std::scoped_lock lock{m_};
            items_.push(std::move(item));
        }
        cv_.notify_one();                       // wake one waiting consumer
    }

    void close()
    {
        {
            std::scoped_lock lock{m_};
            closed_ = true;
        }
        cv_.notify_all();
    }

    std::optional<T> pop()                      // blocks; nullopt when closed and empty
    {
        std::unique_lock lock{m_};
        cv_.wait(lock, [this] { return !items_.empty() || closed_; });   // handles spurious wakeups
        if (items_.empty()) return std::nullopt;
        T item = std::move(items_.front());
        items_.pop();
        return item;
    }

private:
    std::mutex m_;
    std::condition_variable cv_;
    std::queue<T> items_;
    bool closed_ = false;
};

int main()
{
    WorkQueue<int> q;
    std::atomic<long> total{0};
    {
        std::vector<std::jthread> consumers;
        for (int i = 0; i < 3; ++i) {
            consumers.emplace_back([&] {
                while (auto job = q.pop()) total += *job * *job;
            });
        }
        for (int i = 1; i <= 100; ++i) q.push(i);
        q.close();
    }                                           // consumers finish and are joined
    std::cout << total << '\n';                 // 338350 = 1² + 2² + ... + 100²
}
```

Always wait **with a predicate** — `wait` can wake up spuriously, and the predicate is rechecked
under the lock.

### std::async and futures: tasks that return values

```cpp
#include <future>
#include <iostream>
#include <numeric>
#include <vector>

int main()
{
    std::vector<long> data(10'000'000, 1);
    auto mid = data.begin() + data.size() / 2;

    auto first_half = std::async(std::launch::async, [&] { return std::accumulate(data.begin(), mid, 0L); });
    long second = std::accumulate(mid, data.end(), 0L);        // meanwhile, on this thread
    std::cout << first_half.get() + second << '\n';            // get() waits for the result
}
```

`get()` also rethrows any exception thrown in the task. Many standard algorithms also accept an
execution policy — `std::sort(std::execution::par, v.begin(), v.end())` — to parallelize
automatically (with GCC this needs the TBB library installed).

### Stopping threads cooperatively: stop_token

A `jthread` passes a `std::stop_token` to its function if it accepts one; `request_stop()` (called
automatically by the destructor) asks the thread to finish:

```cpp
std::jthread ticker{[](std::stop_token st) {
    while (!st.stop_requested()) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}};
// ... later, or at scope exit:
ticker.request_stop();
```

### C++20 synchronization helpers

- `std::latch` — a one-shot countdown: threads wait until N events have happened.
- `std::barrier` — reusable: N threads wait for each other at the end of each phase.
- `std::counting_semaphore` — limit how many threads use a resource at once.

## Common mistakes

- Sharing data without synchronization ("it worked in testing").
- Manual `lock()`/`unlock()`; forgetting to unlock on an error path.
- Waiting on a condition variable without a predicate.
- Holding a lock while doing slow work or calling callbacks.
- Capturing locals by reference in a thread that outlives them (day 9!).
- Creating a thread per tiny task — threads are expensive; use a fixed pool of workers.
- Expecting more threads to always be faster: memory bandwidth, locks and false sharing limit it.
  Measure.

## Exercises

1. Run the racy counter several times; then with `-fsanitize=thread`. Fix it with a mutex, then with
   `std::atomic<long>`, and time all three versions (without sanitizers, `-O2`).
2. **Parallel sum:** split a vector of 10⁸ numbers into N chunks, sum each chunk in its own
   `jthread` (each writing to its own slot in a results vector — no lock needed, why?), then combine.
   Plot time vs N from 1 to 16.
3. Type in the `WorkQueue`. Use it to count primes in [1, 10⁷] with 4 consumer threads, each taking
   ranges of 10,000 numbers as jobs.
4. Write a **deadlock** on purpose with two mutexes locked in opposite orders by two threads. Then
   fix it with `std::scoped_lock{a, b}`.
5. Use `std::async` to compute the word counts of several text files in parallel (one task per
   file, each returning a map), then merge the maps.
6. Write a `jthread` that prints a progress dot every 200 ms until the main thread finishes a long
   computation and requests stop.
7. ★ Write a simple **thread pool**: N worker `jthread`s pulling `std::function<void()>` jobs from
   a `WorkQueue`, with a `submit` that returns a `std::future` (look up `std::packaged_task`).

## Check yourself

1. What's a data race, and why is it undefined behavior rather than just a wrong result?
2. Why prefer `std::jthread` to `std::thread`?
3. Why use `std::scoped_lock` instead of calling `lock()`/`unlock()`?
4. When is an atomic enough, and when do you need a mutex?
5. Why must `condition_variable::wait` be given a predicate?
