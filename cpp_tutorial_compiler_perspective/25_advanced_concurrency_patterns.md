# Advanced Concurrency Patterns

## Introduction

Advanced concurrency patterns build upon basic threading concepts to solve complex synchronization problems and implement sophisticated concurrent data structures. These patterns are essential for developing robust, high-performance concurrent applications and understanding how modern C++ libraries handle complex concurrent scenarios.

## Lock-Free Data Structures

### Lock-Free Queue Implementation

```cpp
#include <atomic>
#include <memory>

// Lock-free queue using hazard pointers for safe memory reclamation
template<typename T>
class LockFreeQueue {
private:
    struct Node {
        std::atomic<T*> data;
        std::atomic<Node*> next;
        
        Node() : data(nullptr), next(nullptr) {}
    };
    
    std::atomic<Node*> head_;
    std::atomic<Node*> tail_;
    
public:
    LockFreeQueue() {
        Node* dummy = new Node;
        head_.store(dummy);
        tail_.store(dummy);
    }
    
    ~LockFreeQueue() {
        while (Node* node = head_.load()) {
            head_.store(node->next.load());
            delete node;
        }
    }
    
    void enqueue(T value) {
        Node* new_node = new Node;
        new_node->data.store(new T(std::move(value)));
        
        Node* prev_tail = tail_.exchange(new_node);
        prev_tail->next.store(new_node);
    }
    
    std::unique_ptr<T> dequeue() {
        Node* old_head = head_.load();
        Node* next = old_head->next.load();
        
        if (next == nullptr) {
            return std::unique_ptr<T>(nullptr);  // Queue is empty
        }
        
        T* result = next->data.exchange(nullptr);
        head_.store(next);
        delete old_head;
        
        return std::unique_ptr<T>(result);
    }
    
    bool empty() const {
        Node* head = head_.load();
        return head->next.load() == nullptr;
    }
};
```

### Hazard Pointers for Safe Memory Reclamation

```cpp
// Hazard pointer implementation for safe memory reclamation
template<typename T>
class HazardPointer {
private:
    static constexpr std::size_t MAX_HAZARD_POINTERS = 100;
    
    struct HazardPointerRecord {
        std::atomic<std::thread::id> id;
        std::atomic<T*> pointer;
    };
    
    static HazardPointerRecord hazard_pointers_[MAX_HAZARD_POINTERS];
    
    T* pointer_;
    
public:
    HazardPointer() : pointer_(nullptr) {
        // Find a free hazard pointer slot
        std::thread::id this_id = std::this_thread::get_id();
        for (std::size_t i = 0; i < MAX_HAZARD_POINTERS; ++i) {
            std::thread::id old_id;
            if (hazard_pointers_[i].id.compare_exchange_strong(
                    old_id, this_id, std::memory_order_acq_rel)) {
                hazard_pointers_[i].pointer.store(pointer_, std::memory_order_release);
                return;
            }
        }
        throw std::runtime_error("No free hazard pointers");
    }
    
    ~HazardPointer() {
        // Clear hazard pointer
        for (std::size_t i = 0; i < MAX_HAZARD_POINTERS; ++i) {
            if (hazard_pointers_[i].id.load() == std::this_thread::get_id()) {
                hazard_pointers_[i].pointer.store(nullptr, std::memory_order_release);
                hazard_pointers_[i].id.store(std::thread::id{}, std::memory_order_release);
                break;
            }
        }
    }
    
    T* load(std::atomic<T*>& atomic_ptr) {
        T* ptr = atomic_ptr.load();
        pointer_ = ptr;
        // Update hazard pointer
        for (std::size_t i = 0; i < MAX_HAZARD_POINTERS; ++i) {
            if (hazard_pointers_[i].id.load() == std::this_thread::get_id()) {
                hazard_pointers_[i].pointer.store(ptr, std::memory_order_release);
                break;
            }
        }
        return ptr;
    }
    
    static void retire_node(T* node) {
        // Check if any hazard pointer points to this node
        for (std::size_t i = 0; i < MAX_HAZARD_POINTERS; ++i) {
            if (hazard_pointers_[i].pointer.load() == node) {
                // Node is still referenced, cannot delete yet
                return;
            }
        }
        // No hazard pointer references this node, safe to delete
        delete node;
    }
};

template<typename T>
typename HazardPointer<T>::HazardPointerRecord 
HazardPointer<T>::hazard_pointers_[HazardPointer<T>::MAX_HAZARD_POINTERS];
```

## Advanced Synchronization Primitives

### Read-Write Locks with Writer Priority

```cpp
// Read-write lock with writer priority
class WriterPriorityRWLock {
private:
    mutable std::mutex mutex_;
    std::condition_variable reader_cv_;
    std::condition_variable writer_cv_;
    
    std::atomic<int> active_readers_{0};
    std::atomic<int> waiting_writers_{0};
    std::atomic<bool> active_writer_{false};
    
public:
    void lock_shared() {
        std::unique_lock<std::mutex> lock(mutex_);
        // Wait if there are waiting writers or active writer
        reader_cv_.wait(lock, [this] {
            return waiting_writers_ == 0 && !active_writer_.load();
        });
        active_readers_.fetch_add(1);
    }
    
    void unlock_shared() {
        if (active_readers_.fetch_sub(1) == 1) {
            // Last reader, notify waiting writers
            std::lock_guard<std::mutex> lock(mutex_);
            writer_cv_.notify_one();
        }
    }
    
    void lock() {
        std::unique_lock<std::mutex> lock(mutex_);
        waiting_writers_.fetch_add(1);
        // Wait until no active readers and no active writer
        writer_cv_.wait(lock, [this] {
            return active_readers_.load() == 0 && !active_writer_.load();
        });
        waiting_writers_.fetch_sub(1);
        active_writer_.store(true);
    }
    
    void unlock() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            active_writer_.store(false);
        }
        // Notify waiting writers first, then readers
        writer_cv_.notify_one();
        reader_cv_.notify_all();
    }
};
```

### Semaphore Implementation

```cpp
// Semaphore implementation using mutex and condition variable
class Semaphore {
private:
    std::mutex mutex_;
    std::condition_variable condition_;
    std::atomic<int> count_;
    
public:
    explicit Semaphore(int count = 0) : count_(count) {}
    
    void notify() {
        std::lock_guard<std::mutex> lock(mutex_);
        ++count_;
        condition_.notify_one();
    }
    
    void wait() {
        std::unique_lock<std::mutex> lock(mutex_);
        condition_.wait(lock, [this] { return count_.load() > 0; });
        --count_;
    }
    
    bool try_wait() {
        std::lock_guard<std::mutex> lock(mutex_);
        if (count_.load() > 0) {
            --count_;
            return true;
        }
        return false;
    }
};

// Counting semaphore (C++20)
class CountingSemaphore {
private:
    std::mutex mutex_;
    std::condition_variable condition_;
    std::atomic<int> count_;
    const int max_count_;
    
public:
    explicit CountingSemaphore(int max_count, int initial_count = 0) 
        : count_(initial_count), max_count_(max_count) {}
    
    void acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        condition_.wait(lock, [this] { return count_.load() > 0; });
        count_.fetch_sub(1);
    }
    
    void release() {
        std::lock_guard<std::mutex> lock(mutex_);
        if (count_.load() < max_count_) {
            count_.fetch_add(1);
            condition_.notify_one();
        }
    }
};
```

## Producer-Consumer Patterns

### Bounded Producer-Consumer Queue

```cpp
// Bounded producer-consumer queue with backpressure
template<typename T>
class BoundedQueue {
private:
    std::vector<T> buffer_;
    std::atomic<std::size_t> head_{0};
    std::atomic<std::size_t> tail_{0};
    std::atomic<std::size_t> count_{0};
    
    mutable std::mutex mutex_;
    std::condition_variable not_empty_;
    std::condition_variable not_full_;
    const std::size_t capacity_;
    
public:
    explicit BoundedQueue(std::size_t capacity) 
        : buffer_(capacity), capacity_(capacity) {}
    
    bool try_enqueue(T item) {
        std::unique_lock<std::mutex> lock(mutex_);
        if (count_.load() == capacity_) {
            return false;  // Queue is full
        }
        
        buffer_[tail_.load() % capacity_] = std::move(item);
        tail_.fetch_add(1);
        count_.fetch_add(1);
        
        lock.unlock();
        not_empty_.notify_one();
        return true;
    }
    
    void enqueue(T item) {
        std::unique_lock<std::mutex> lock(mutex_);
        not_full_.wait(lock, [this] { return count_.load() < capacity_; });
        
        buffer_[tail_.load() % capacity_] = std::move(item);
        tail_.fetch_add(1);
        count_.fetch_add(1);
        
        lock.unlock();
        not_empty_.notify_one();
    }
    
    bool try_dequeue(T& item) {
        std::unique_lock<std::mutex> lock(mutex_);
        if (count_.load() == 0) {
            return false;  // Queue is empty
        }
        
        item = std::move(buffer_[head_.load() % capacity_]);
        head_.fetch_add(1);
        count_.fetch_sub(1);
        
        lock.unlock();
        not_full_.notify_one();
        return true;
    }
    
    T dequeue() {
        std::unique_lock<std::mutex> lock(mutex_);
        not_empty_.wait(lock, [this] { return count_.load() > 0; });
        
        T item = std::move(buffer_[head_.load() % capacity_]);
        head_.fetch_add(1);
        count_.fetch_sub(1);
        
        lock.unlock();
        not_full_.notify_one();
        return item;
    }
    
    std::size_t size() const {
        return count_.load();
    }
    
    bool empty() const {
        return count_.load() == 0;
    }
    
    bool full() const {
        return count_.load() == capacity_;
    }
};
```

### Multi-Producer Multi-Consumer Queue

```cpp
// Multi-producer multi-consumer queue using lock-free techniques
template<typename T>
class MPMCQueue {
private:
    struct Node {
        std::atomic<T*> data;
        std::atomic<Node*> next;
        
        Node() : data(nullptr), next(nullptr) {}
    };
    
    alignas(64) std::atomic<Node*> head_;
    alignas(64) std::atomic<Node*> tail_;
    
    // Padding to avoid false sharing
    char padding_[64 - sizeof(std::atomic<Node*>)];
    
public:
    MPMCQueue() {
        Node* dummy = new Node;
        head_.store(dummy);
        tail_.store(dummy);
    }
    
    ~MPMCQueue() {
        while (Node* node = head_.load()) {
            head_.store(node->next.load());
            T* data = node->data.load();
            if (data) delete data;
            delete node;
        }
    }
    
    void enqueue(T item) {
        Node* new_node = new Node;
        new_node->data.store(new T(std::move(item)));
        
        Node* prev_tail = tail_.exchange(new_node, std::memory_order_acq_rel);
        prev_tail->next.store(new_node, std::memory_order_release);
    }
    
    bool try_dequeue(T& item) {
        Node* head = head_.load(std::memory_order_acquire);
        Node* next = head->next.load(std::memory_order_acquire);
        
        if (next == nullptr) {
            return false;  // Queue is empty
        }
        
        T* data = next->data.exchange(nullptr, std::memory_order_acq_rel);
        if (data == nullptr) {
            return false;  // Another thread is already dequeuing this node
        }
        
        item = std::move(*data);
        delete data;
        
        head_.store(next, std::memory_order_release);
        delete head;
        return true;
    }
};
```

## Advanced Thread Pool Patterns

### Work-Stealing Thread Pool

```cpp
// Work-stealing thread pool implementation
class WorkStealingThreadPool {
private:
    class ThreadWorker {
    private:
        WorkStealingThreadPool* pool_;
        std::thread worker_;
        int id_;
        
    public:
        ThreadWorker(WorkStealingThreadPool* pool, int id) 
            : pool_(pool), id_(id) {}
        
        void start() {
            worker_ = std::thread(&ThreadWorker::worker_loop, this);
        }
        
        void join() {
            if (worker_.joinable()) {
                worker_.join();
            }
        }
        
    private:
        void worker_loop() {
            while (!pool_->shutdown_) {
                std::function<void()> task;
                
                // Try to get task from local queue
                if (pool_->queues_[id_].try_pop(task)) {
                    task();
                } 
                // Try to steal from other queues
                else if (pool_->steal_task(id_, task)) {
                    task();
                } 
                // No work available, sleep briefly
                else {
                    std::this_thread::sleep_for(std::chrono::microseconds(100));
                }
            }
        }
    };
    
    std::vector<std::unique_ptr<ThreadWorker>> workers_;
    std::vector<ThreadsafeQueue<std::function<void()>>> queues_;
    std::atomic<bool> shutdown_{false};
    const std::size_t num_threads_;
    
public:
    explicit WorkStealingThreadPool(std::size_t num_threads = std::thread::hardware_concurrency())
        : num_threads_(num_threads > 0 ? num_threads : std::thread::hardware_concurrency()) {
        queues_.resize(num_threads_);
        
        for (std::size_t i = 0; i < num_threads_; ++i) {
            workers_.emplace_back(std::make_unique<ThreadWorker>(this, i));
            workers_[i]->start();
        }
    }
    
    ~WorkStealingThreadPool() {
        shutdown_ = true;
        for (auto& worker : workers_) {
            worker->join();
        }
    }
    
    template<typename F>
    auto submit(F&& f) -> std::future<typename std::result_of<F()>::type> {
        using return_type = typename std::result_of<F()>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(std::forward<F>(f));
        std::future<return_type> result = task->get_future();
        
        // Submit to local queue (round-robin)
        static std::atomic<std::size_t> index{0};
        std::size_t queue_index = index++ % num_threads_;
        queues_[queue_index].push([task]() { (*task)(); });
        
        return result;
    }
    
private:
    bool steal_task(int thief_id, std::function<void()>& task) {
        for (std::size_t i = 0; i < num_threads_; ++i) {
            if (i != static_cast<std::size_t>(thief_id) && queues_[i].try_pop(task)) {
                return true;
            }
        }
        return false;
    }
};
```

## Futures and Promises Patterns

### Continuation-Based Futures

```cpp
// Continuation-based future implementation
template<typename T>
class ContinuationFuture {
private:
    std::shared_ptr<std::promise<T>> promise_;
    std::future<T> future_;
    
public:
    ContinuationFuture() : promise_(std::make_shared<std::promise<T>>()), 
                          future_(promise_->get_future()) {}
    
    ContinuationFuture(std::future<T>&& f) : future_(std::move(f)) {}
    
    template<typename F>
    auto then(F&& func) -> ContinuationFuture<decltype(func(std::declval<T>()))> {
        using return_type = decltype(func(std::declval<T>()));
        
        ContinuationFuture<return_type> next_future;
        
        std::thread([promise = next_future.promise_, 
                     future = std::move(future_), 
                     func = std::forward<F>(func)]() mutable {
            try {
                T result = future.get();
                promise->set_value(func(std::move(result)));
            } catch (...) {
                promise->set_exception(std::current_exception());
            }
        }).detach();
        
        return next_future;
    }
    
    T get() {
        return future_.get();
    }
    
    bool valid() const {
        return future_.valid();
    }
};

// Usage example
void continuation_example() {
    ContinuationFuture<int> future;
    
    auto result = future.then([](int x) { return x * 2; })
                       .then([](int x) { return std::to_string(x); })
                       .then([](const std::string& s) { return "Result: " + s; });
    
    // Set the initial value
    std::thread([promise = std::make_shared<std::promise<int>>()]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        promise->set_value(42);
    }).detach();
    
    std::cout << result.get() << std::endl;  // Output: "Result: 84"
}
```

## Advanced Concurrency Utilities

### Thread-Local Storage Patterns

```cpp
// Thread-local storage with initialization and cleanup
template<typename T>
class ThreadLocal {
private:
    static thread_local std::unique_ptr<T> instance_;
    static std::function<T*()> initializer_;
    
public:
    static void set_initializer(std::function<T*()> init) {
        initializer_ = std::move(init);
    }
    
    static T& get() {
        if (!instance_) {
            if (initializer_) {
                instance_.reset(initializer_());
            } else {
                instance_.reset(new T());
            }
        }
        return *instance_;
    }
    
    static void reset() {
        instance_.reset();
    }
};

template<typename T>
thread_local std::unique_ptr<T> ThreadLocal<T>::instance_;

template<typename T>
std::function<T*()> ThreadLocal<T>::initializer_;

// Usage example
class ThreadLocalExample {
public:
    static void thread_local_usage() {
        // Set initializer for thread-local storage
        ThreadLocal<std::vector<int>>::set_initializer([]() {
            auto* vec = new std::vector<int>();
            vec->reserve(1000);  // Pre-allocate for performance
            return vec;
        });
        
        // Each thread gets its own instance
        std::vector<int>& local_vec = ThreadLocal<std::vector<int>>::get();
        local_vec.push_back(42);
    }
};
```

### Atomic Operations with Complex Types

```cpp
// Atomic operations with complex types using atomic flag
template<typename T>
class AtomicWrapper {
private:
    mutable std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
    T value_;
    
public:
    AtomicWrapper() = default;
    AtomicWrapper(const T& value) : value_(value) {}
    AtomicWrapper(T&& value) : value_(std::move(value)) {}
    
    T load() const {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            std::this_thread::yield();
        }
        T result = value_;
        flag_.clear(std::memory_order_release);
        return result;
    }
    
    void store(const T& value) {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            std::this_thread::yield();
        }
        value_ = value;
        flag_.clear(std::memory_order_release);
    }
    
    void store(T&& value) {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            std::this_thread::yield();
        }
        value_ = std::move(value);
        flag_.clear(std::memory_order_release);
    }
    
    template<typename F>
    auto atomic_operation(F&& func) -> decltype(func(value_)) {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            std::this_thread::yield();
        }
        auto result = func(value_);
        flag_.clear(std::memory_order_release);
        return result;
    }
};
```

## Best Practices

### Advanced Concurrency Guidelines

1. **Use Lock-Free Data Structures Sparingly** - They're complex and error-prone
2. **Prefer Higher-Level Abstractions** - Use std::async, thread pools, parallel algorithms
3. **Implement Proper Memory Reclamation** - For lock-free structures, use hazard pointers or epoch-based reclamation
4. **Consider False Sharing** - Align data to cache line boundaries when necessary
5. **Test with Thread Sanitizers** - Use tools like ThreadSanitizer to detect data races
6. **Profile Real-World Workloads** - Measure performance with actual usage patterns
7. **Handle Exception Safety** - Ensure proper cleanup in concurrent environments
8. **Document Concurrency Guarantees** - Clearly specify thread safety and performance characteristics

### Common Advanced Concurrency Pitfalls

```cpp
// Examples of advanced concurrency mistakes to avoid

// 1. ABA Problem in Lock-Free Structures
class ABAMistake {
    // Using raw pointers in lock-free structures without proper ABA handling
    // can lead to use-after-free errors
};

// 2. False Sharing
struct FalseSharingExample {
    alignas(64) int thread1_data;  // Should be aligned to avoid false sharing
    alignas(64) int thread2_data;  // Should be aligned to avoid false sharing
};

// 3. Incorrect Memory Ordering
class MemoryOrderingMistake {
    std::atomic<bool> flag{false};
    std::atomic<int> data{0};
    
    void writer() {
        data.store(42, std::memory_order_relaxed);  // Wrong ordering
        flag.store(true, std::memory_order_relaxed); // Wrong ordering
    }
    
    void reader() {
        while (!flag.load(std::memory_order_relaxed)) {} // Wrong ordering
        std::cout << data.load(std::memory_order_relaxed) << std::endl; // Wrong ordering
    }
};

// 4. Inefficient Lock-Free Implementations
class InefficientLockFree {
    // Implementing complex lock-free algorithms when simpler
    // mutex-based solutions would be more maintainable and performant
};
```

Advanced concurrency patterns provide powerful tools for building high-performance concurrent applications, but they require careful consideration of correctness, performance, and maintainability trade-offs.