# Concurrency and Parallelism

## Introduction

Concurrency and parallelism are essential aspects of modern C++ programming that enable applications to perform multiple tasks simultaneously, taking advantage of multi-core processors. Understanding these concepts is crucial for developing high-performance applications and for comprehending how the STL handles concurrent operations.

## Threading Fundamentals

### Basic Thread Creation and Management

```cpp
#include <thread>
#include <mutex>
#include <future>
#include <atomic>

// Basic thread creation
void basic_threading_example() {
    // Simple thread creation
    std::thread t1([]() {
        std::cout << "Hello from thread 1!" << std::endl;
    });
    
    // Thread with parameters
    std::thread t2([](int value) {
        std::cout << "Value from thread 2: " << value << std::endl;
    }, 42);
    
    // Join threads to ensure completion
    t1.join();
    t2.join();
}

// Thread with return value using std::future
std::future<int> async_computation() {
    return std::async(std::launch::async, []() -> int {
        // Simulate some computation
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        return 42 * 42;
    });
}

// Thread-safe counter
class ThreadSafeCounter {
private:
    mutable std::mutex mutex_;
    int value_;
    
public:
    ThreadSafeCounter() : value_(0) {}
    
    void increment() {
        std::lock_guard<std::mutex> lock(mutex_);
        ++value_;
    }
    
    int get() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return value_;
    }
};
```

### RAII Thread Management

```cpp
// RAII wrapper for thread management
class ThreadRAII {
private:
    std::thread thread_;
    bool join_on_destruction_;
    
public:
    explicit ThreadRAII(std::thread t, bool join_on_destruction = true)
        : thread_(std::move(t)), join_on_destruction_(join_on_destruction) {}
    
    ~ThreadRAII() {
        if (thread_.joinable()) {
            if (join_on_destruction_) {
                thread_.join();
            } else {
                thread_.detach();
            }
        }
    }
    
    std::thread& get_thread() { return thread_; }
    const std::thread& get_thread() const { return thread_; }
    
    std::thread release() {
        join_on_destruction_ = false;
        return std::move(thread_);
    }
};

// Usage example
void raii_threading_example() {
    ThreadRAII t(std::thread([]() {
        std::cout << "RAII thread execution" << std::endl;
    }));
    
    // Thread automatically joined when t goes out of scope
}
```

## Synchronization Primitives

### Mutexes and Locking

```cpp
#include <shared_mutex>

// Different types of mutexes for different scenarios
class SynchronizationExamples {
public:
    // Exclusive mutex
    void exclusive_mutex_example() {
        std::mutex mtx;
        
        std::thread writer([&mtx]() {
            std::lock_guard<std::mutex> lock(mtx);
            // Critical section
            std::cout << "Writing data..." << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        });
        
        std::thread reader([&mtx]() {
            std::lock_guard<std::mutex> lock(mtx);
            // Critical section
            std::cout << "Reading data..." << std::endl;
        });
        
        writer.join();
        reader.join();
    }
    
    // Reader-writer mutex (C++17)
    void shared_mutex_example() {
        std::shared_mutex rw_mtx;
        std::vector<int> data{1, 2, 3, 4, 5};
        
        // Multiple readers can access simultaneously
        std::thread reader1([&rw_mtx, &data]() {
            std::shared_lock<std::shared_mutex> lock(rw_mtx);
            std::cout << "Reader 1: ";
            for (int val : data) std::cout << val << " ";
            std::cout << std::endl;
        });
        
        std::thread reader2([&rw_mtx, &data]() {
            std::shared_lock<std::shared_mutex> lock(rw_mtx);
            std::cout << "Reader 2: ";
            for (int val : data) std::cout << val << " ";
            std::cout << std::endl;
        });
        
        // Writer waits for all readers to finish
        std::thread writer([&rw_mtx, &data]() {
            std::unique_lock<std::shared_mutex> lock(rw_mtx);
            data.push_back(42);
            std::cout << "Data written" << std::endl;
        });
        
        reader1.join();
        reader2.join();
        writer.join();
    }
    
    // Recursive mutex
    void recursive_mutex_example() {
        std::recursive_mutex r_mtx;
        
        std::function<void(int)> recursive_func = [&](int depth) {
            if (depth <= 0) return;
            
            std::lock_guard<std::recursive_mutex> lock(r_mtx);
            std::cout << "Depth: " << depth << std::endl;
            recursive_func(depth - 1);  // Can lock again in the same thread
        };
        
        std::thread t([&recursive_func]() {
            recursive_func(5);
        });
        
        t.join();
    }
};
```

### Condition Variables

```cpp
#include <condition_variable>

// Producer-consumer pattern with condition variables
template<typename T>
class ThreadsafeQueue {
private:
    mutable std::mutex mutex_;
    std::condition_variable condition_;
    std::queue<T> queue_;
    
public:
    void push(T item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            queue_.push(std::move(item));
        }
        condition_.notify_one();
    }
    
    T pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        condition_.wait(lock, [this] { return !queue_.empty(); });
        
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }
    
    bool try_pop(T& item) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) {
            return false;
        }
        
        item = std::move(queue_.front());
        queue_.pop();
        return true;
    }
    
    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.empty();
    }
};

// Example usage
void producer_consumer_example() {
    ThreadsafeQueue<int> queue;
    
    // Producer thread
    std::thread producer([&queue]() {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    });
    
    // Consumer thread
    std::thread consumer([&queue]() {
        for (int i = 0; i < 10; ++i) {
            int value = queue.pop();
            std::cout << "Consumed: " << value << std::endl;
        }
    });
    
    producer.join();
    consumer.join();
}
```

## Atomic Operations

### Atomic Types and Operations

```cpp
// Atomic operations for lock-free programming
class AtomicOperations {
public:
    // Basic atomic operations
    void basic_atomic_example() {
        std::atomic<int> atomic_counter{0};
        
        // Multiple threads incrementing
        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back([&atomic_counter]() {
                for (int j = 0; j < 1000; ++j) {
                    atomic_counter.fetch_add(1);
                }
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
        
        std::cout << "Final counter value: " << atomic_counter.load() << std::endl;
    }
    
    // Atomic flags
    void atomic_flag_example() {
        std::atomic_flag lock = ATOMIC_FLAG_INIT;
        
        auto critical_section = [&lock]() {
            // Spin until we acquire the flag
            while (lock.test_and_set(std::memory_order_acquire)) {
                std::this_thread::yield();  // Yield to other threads
            }
            
            // Critical section
            std::cout << "In critical section" << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            
            // Release the flag
            lock.clear(std::memory_order_release);
        };
        
        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back(critical_section);
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
    
    // Atomic compare-and-swap
    void atomic_cas_example() {
        std::atomic<int> atomic_value{0};
        
        // Atomic increment using compare-and-swap
        auto atomic_increment = [&atomic_value]() {
            int expected = atomic_value.load();
            while (!atomic_value.compare_exchange_weak(expected, expected + 1)) {
                // expected is updated with current value on failure
                // weak version may fail spuriously, so we loop
            }
        };
        
        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back(atomic_increment);
        }
        
        for (auto& t : threads) {
            t.join();
        }
        
        std::cout << "Final value: " << atomic_value.load() << std::endl;
    }
};
```

### Memory Ordering

```cpp
// Memory ordering for atomic operations
class MemoryOrdering {
public:
    // Different memory ordering levels
    void memory_ordering_example() {
        std::atomic<bool> ready{false};
        std::atomic<int> data{0};
        
        std::thread writer([&ready, &data]() {
            data.store(42, std::memory_order_relaxed);  // Store data
            ready.store(true, std::memory_order_release);  // Signal ready
        });
        
        std::thread reader([&ready, &data]() {
            while (!ready.load(std::memory_order_acquire)) {  // Wait for ready
                std::this_thread::yield();
            }
            // Load is synchronized with store in writer
            std::cout << "Data: " << data.load(std::memory_order_relaxed) << std::endl;
        });
        
        writer.join();
        reader.join();
    }
    
    // Sequential consistency (default)
    void sequential_consistency_example() {
        std::atomic<int> x{0}, y{0};
        std::atomic<int> r1{0}, r2{0};
        
        std::thread t1([&x, &y, &r1, &r2]() {
            x.store(1, std::memory_order_seq_cst);
            r1.store(y.load(std::memory_order_seq_cst), std::memory_order_relaxed);
        });
        
        std::thread t2([&x, &y, &r1, &r2]() {
            y.store(1, std::memory_order_seq_cst);
            r2.store(x.load(std::memory_order_seq_cst), std::memory_order_relaxed);
        });
        
        t1.join();
        t2.join();
        
        // With sequential consistency, we cannot have r1 = 1 and r2 = 1
        std::cout << "r1: " << r1.load() << ", r2: " << r2.load() << std::endl;
    }
};
```

## Parallel Algorithms

### STL Parallel Algorithms (C++17)

```cpp
#include <execution>
#include <algorithm>
#include <numeric>

// Using parallel algorithms from STL
class ParallelAlgorithms {
public:
    // Parallel execution policies
    void parallel_execution_policies() {
        std::vector<int> data(1000000);
        std::iota(data.begin(), data.end(), 1);
        
        // Sequential execution
        auto sum_seq = std::accumulate(data.begin(), data.end(), 0LL);
        
        // Parallel execution
        auto sum_par = std::reduce(std::execution::par, data.begin(), data.end(), 0LL);
        
        // Parallel + vectorized execution
        auto sum_par_unseq = std::reduce(std::execution::par_unseq, 
                                        data.begin(), data.end(), 0LL);
        
        std::cout << "Sequential: " << sum_seq << std::endl;
        std::cout << "Parallel: " << sum_par << std::endl;
        std::cout << "Parallel + Unsequenced: " << sum_par_unseq << std::endl;
    }
    
    // Parallel transform
    void parallel_transform() {
        std::vector<double> input(1000000);
        std::iota(input.begin(), input.end(), 1.0);
        std::vector<double> output(input.size());
        
        // Parallel transformation
        std::transform(std::execution::par_unseq,
                      input.begin(), input.end(),
                      output.begin(),
                      [](double x) { return std::sqrt(x * x + 1); });
    }
    
    // Parallel sort
    void parallel_sort() {
        std::vector<int> data(1000000);
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_int_distribution<> dis(1, 1000000);
        
        for (auto& val : data) {
            val = dis(gen);
        }
        
        auto start = std::chrono::high_resolution_clock::now();
        std::sort(std::execution::par_unseq, data.begin(), data.end());
        auto end = std::chrono::high_resolution_clock::now();
        
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "Parallel sort took: " << duration.count() << " ms" << std::endl;
    }
};
```

### Custom Parallel Algorithms

```cpp
// Implementing custom parallel algorithms
template<typename Iterator, typename Function>
void parallel_for_each(Iterator first, Iterator last, Function f) {
    const std::size_t min_per_thread = 256;
    const std::size_t max_threads = 
        (last - first + min_per_thread - 1) / min_per_thread;
    
    const std::size_t hardware_threads = std::thread::hardware_concurrency();
    const std::size_t num_threads = 
        std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
    
    const std::size_t block_size = (last - first) / num_threads;
    
    std::vector<std::future<void>> futures;
    std::vector<std::thread> threads;
    
    Iterator block_start = first;
    for (unsigned i = 0; i < num_threads; ++i) {
        Iterator block_end = (i == num_threads - 1) ? last : block_start + block_size;
        
        std::packaged_task<void()> task([block_start, block_end, f]() {
            std::for_each(block_start, block_end, f);
        });
        
        futures.push_back(task.get_future());
        threads.emplace_back(std::move(task));
        
        block_start = block_end;
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
    
    for (auto& future : futures) {
        future.wait();
    }
}

// Parallel reduction
template<typename Iterator, typename T>
T parallel_reduce(Iterator first, Iterator last, T init) {
    const std::size_t min_per_thread = 256;
    const std::size_t max_threads = 
        (last - first + min_per_thread - 1) / min_per_thread;
    
    const std::size_t hardware_threads = std::thread::hardware_concurrency();
    const std::size_t num_threads = 
        std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
    
    if (num_threads < 2) {
        return std::accumulate(first, last, init);
    }
    
    const std::size_t block_size = (last - first) / num_threads;
    
    std::vector<std::thread> threads;
    std::vector<T> results(num_threads);
    
    Iterator block_start = first;
    for (unsigned i = 0; i < num_threads; ++i) {
        Iterator block_end = (i == num_threads - 1) ? last : block_start + block_size;
        
        threads.emplace_back([block_start, block_end, &results, i]() {
            results[i] = std::accumulate(block_start, block_end, T{});
        });
        
        block_start = block_end;
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
    
    return std::accumulate(results.begin(), results.end(), init);
}
```

## Advanced Concurrency Patterns

### Lock-Free Data Structures

```cpp
// Simple lock-free stack implementation
template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        
        Node(T const& data_) : data(data_) {}
    };
    
    std::atomic<Node*> head_{nullptr};
    
public:
    void push(T const& data) {
        Node* new_node = new Node(data);
        new_node->next = head_.load();
        
        // Atomic compare-and-swap to update head
        while (!head_.compare_exchange_weak(new_node->next, new_node)) {
            // Update new_node->next with the current head value
            // and try again
        }
    }
    
    std::shared_ptr<T> pop() {
        Node* old_head = head_.load();
        
        while (old_head && 
               !head_.compare_exchange_weak(old_head, old_head->next)) {
            // Update old_head with the current head value
            // and try again
        }
        
        return old_head ? std::shared_ptr<T>(std::make_shared<T>(old_head->data)) : std::shared_ptr<T>();
    }
    
    bool empty() const {
        return head_.load() == nullptr;
    }
};
```

### Thread Pools

```cpp
// Simple thread pool implementation
class ThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    
    std::mutex queue_mutex_;
    std::condition_variable condition_;
    bool stop_;
    
public:
    explicit ThreadPool(std::size_t num_threads) : stop_(false) {
        for (std::size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex_);
                        condition_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                        
                        if (stop_ && tasks_.empty()) {
                            return;
                        }
                        
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    
                    task();
                }
            });
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type> {
        using return_type = typename std::result_of<F(Args...)>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> res = task->get_future();
        
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            
            if (stop_) {
                throw std::runtime_error("enqueue on stopped ThreadPool");
            }
            
            tasks_.emplace([task]() { (*task)(); });
        }
        
        condition_.notify_one();
        return res;
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            stop_ = true;
        }
        
        condition_.notify_all();
        
        for (std::thread& worker : workers_) {
            worker.join();
        }
    }
};

// Usage example
void thread_pool_example() {
    ThreadPool pool(4);  // 4 worker threads
    
    std::vector<std::future<int>> results;
    
    for (int i = 0; i < 8; ++i) {
        results.emplace_back(pool.enqueue([i] {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            return i * i;
        }));
    }
    
    for (auto& result : results) {
        std::cout << result.get() << " ";
    }
    std::cout << std::endl;
}
```

## Exception Safety in Concurrent Code

### Handling Exceptions in Multithreaded Environments

```cpp
// Exception safety in concurrent code
class ConcurrentExceptionSafety {
public:
    // Proper exception handling in threads
    void thread_exception_handling() {
        std::vector<std::thread> threads;
        
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back([i]() {
                try {
                    if (i == 2) {
                        throw std::runtime_error("Exception in thread 2");
                    }
                    std::cout << "Thread " << i << " completed successfully" << std::endl;
                } catch (const std::exception& e) {
                    std::cerr << "Exception in thread " << i << ": " << e.what() << std::endl;
                    // Handle exception appropriately
                }
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
    
    // Exception safety with futures
    std::future<void> async_exception_example() {
        return std::async(std::launch::async, []() {
            throw std::runtime_error("Exception from async task");
        });
    }
    
    // Safe exception propagation
    void safe_exception_propagation() {
        std::vector<std::future<void>> futures;
        
        for (int i = 0; i < 4; ++i) {
            futures.push_back(std::async(std::launch::async, [i]() {
                if (i == 2) {
                    throw std::runtime_error("Error in async task");
                }
            }));
        }
        
        for (auto& future : futures) {
            try {
                future.wait();
                future.get();  // This will re-throw any exception
            } catch (const std::exception& e) {
                std::cerr << "Caught exception: " << e.what() << std::endl;
            }
        }
    }
};
```

## Best Practices

### Concurrency Guidelines

1. **Start Simple** - Begin with basic threading and synchronization
2. **Prefer Higher-Level Abstractions** - Use std::async, parallel algorithms, thread pools
3. **Minimize Shared State** - Reduce contention and synchronization overhead
4. **Consider Lock-Free Alternatives** - For performance-critical sections
5. **Test Thoroughly** - Concurrency bugs are often intermittent
6. **Profile Performance** - Measure actual performance gains
7. **Handle Exceptions** - Ensure proper exception handling in concurrent code
8. **Document Thread Safety** - Clearly specify thread safety of interfaces

### Common Concurrency Pitfalls

```cpp
// Examples of common concurrency mistakes to avoid

// 1. Data races
class BadDataRace {
    int value_ = 0;
public:
    void increment() {
        // This is not thread-safe!
        ++value_;  // Multiple threads can interfere
    }
};

// 2. Deadlocks
void deadlock_example() {
    std::mutex m1, m2;
    
    std::thread t1([&m1, &m2]() {
        std::lock_guard<std::mutex> lock1(m1);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::lock_guard<std::mutex> lock2(m2);  // May deadlock
    });
    
    std::thread t2([&m1, &m2]() {
        std::lock_guard<std::mutex> lock2(m2);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::lock_guard<std::mutex> lock1(m1);  // May deadlock
    });
    
    // Solution: Use std::lock to lock multiple mutexes atomically
    // std::lock(m1, m2);
    // std::lock_guard<std::mutex> lock1(m1, std::adopt_lock);
    // std::lock_guard<std::mutex> lock2(m2, std::adopt_lock);
}

// 3. ABA problems with lock-free structures
// Use std::atomic<T>::compare_exchange_weak/strong with proper ABA handling
```

Understanding concurrency and parallelism is crucial for developing high-performance C++ applications that can fully utilize modern multi-core processors.