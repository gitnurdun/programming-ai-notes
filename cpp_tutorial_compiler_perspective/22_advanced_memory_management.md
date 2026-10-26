# Advanced Memory Management

## Introduction

Advanced memory management techniques in C++ go beyond basic allocation and deallocation to include sophisticated strategies for optimizing performance, reducing fragmentation, and ensuring efficient resource utilization. These techniques are essential for developing high-performance applications and understanding how modern C++ libraries manage memory internally.

## Memory Allocators Deep Dive

### Custom Allocator Design Patterns

```cpp
// Thread-local allocator for improved performance in multi-threaded environments
template<typename T>
class ThreadLocalAllocator {
private:
    static thread_local std::pmr::memory_resource* thread_resource_;
    
public:
    using value_type = T;
    
    ThreadLocalAllocator() = default;
    
    template<typename U>
    ThreadLocalAllocator(const ThreadLocalAllocator<U>&) noexcept {}
    
    T* allocate(std::size_t n) {
        if (!thread_resource_) {
            thread_resource_ = std::pmr::new_delete_resource();
        }
        
        return static_cast<T*>(thread_resource_->allocate(n * sizeof(T), alignof(T)));
    }
    
    void deallocate(T* ptr, std::size_t n) {
        thread_resource_->deallocate(ptr, n * sizeof(T), alignof(T));
    }
    
    template<typename U>
    bool operator==(const ThreadLocalAllocator<U>&) const noexcept {
        return true;
    }
    
    template<typename U>
    bool operator!=(const ThreadLocalAllocator<U>&) const noexcept {
        return false;
    }
};

template<typename T>
thread_local std::pmr::memory_resource* ThreadLocalAllocator<T>::thread_resource_ = nullptr;

// Usage example
using ThreadSafeVector = std::vector<int, ThreadLocalAllocator<int>>;
```

### Memory Pool with Object Tracking

```cpp
template<typename T, std::size_t BlockSize = 4096>
class TrackedMemoryPool {
private:
    struct Block {
        alignas(T) char data[sizeof(T)];
        Block* next;
        bool in_use;
    };
    
    struct Pool {
        char* memory_pool;
        Block* blocks;
        std::size_t block_count;
        Pool* next_pool;
        
        Pool(std::size_t size) : next_pool(nullptr) {
            memory_pool = static_cast<char*>(std::malloc(size));
            if (!memory_pool) throw std::bad_alloc();
            
            block_count = size / sizeof(Block);
            blocks = reinterpret_cast<Block*>(memory_pool);
            
            // Initialize all blocks
            for (std::size_t i = 0; i < block_count; ++i) {
                blocks[i].next = (i < block_count - 1) ? &blocks[i + 1] : nullptr;
                blocks[i].in_use = false;
            }
        }
        
        ~Pool() {
            std::free(memory_pool);
        }
    };
    
    Pool* current_pool_;
    Block* free_list_;
    std::atomic<std::size_t> allocated_count_;
    std::atomic<std::size_t> total_count_;
    std::mutex pool_mutex_;
    
public:
    TrackedMemoryPool() 
        : current_pool_(nullptr), free_list_(nullptr), 
          allocated_count_(0), total_count_(0) {}
    
    ~TrackedMemoryPool() {
        while (current_pool_) {
            Pool* next = current_pool_->next_pool;
            delete current_pool_;
            current_pool_ = next;
        }
    }
    
    T* allocate() {
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        if (!free_list_) {
            // Create new pool
            Pool* new_pool = new Pool(BlockSize);
            new_pool->next_pool = current_pool_;
            current_pool_ = new_pool;
            
            // Add new blocks to free list
            for (std::size_t i = 0; i < new_pool->block_count; ++i) {
                new_pool->blocks[i].next = free_list_;
                new_pool->blocks[i].in_use = false;
                free_list_ = &new_pool->blocks[i];
            }
        }
        
        Block* block = free_list_;
        free_list_ = block->next;
        block->in_use = true;
        block->next = nullptr;
        
        ++allocated_count_;
        ++total_count_;
        
        return reinterpret_cast<T*>(block->data);
    }
    
    void deallocate(T* ptr) {
        if (!ptr) return;
        
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        Block* block = reinterpret_cast<Block*>(
            reinterpret_cast<char*>(ptr) - offsetof(Block, data));
        
        block->in_use = false;
        block->next = free_list_;
        free_list_ = block;
        
        --allocated_count_;
    }
    
    std::size_t get_allocated_count() const {
        return allocated_count_.load();
    }
    
    std::size_t get_total_count() const {
        return total_count_.load();
    }
    
    double get_utilization_rate() const {
        if (total_count_.load() == 0) return 0.0;
        return static_cast<double>(allocated_count_.load()) / total_count_.load();
    }
};
```

## Memory Layout Optimization

### Cache-Friendly Data Structures

```cpp
// Structure of Arrays (SoA) vs Array of Structures (AoS)
struct ParticleAoS {
    float x, y, z;      // Position
    float vx, vy, vz;   // Velocity
    float mass;
    bool active;
};

// Cache-friendly Structure of Arrays
class ParticleSystem {
private:
    struct DataArrays {
        std::vector<float> x, y, z;         // Positions
        std::vector<float> vx, vy, vz;      // Velocities
        std::vector<float> mass;            // Masses
        std::vector<bool> active;           // Active flags
    } data_;
    
    std::size_t particle_count_;
    
public:
    void add_particle(float x, float y, float z, 
                     float vx, float vy, float vz, 
                     float mass, bool active = true) {
        data_.x.push_back(x);
        data_.y.push_back(y);
        data_.z.push_back(z);
        data_.vx.push_back(vx);
        data_.vy.push_back(vy);
        data_.vz.push_back(vz);
        data_.mass.push_back(mass);
        data_.active.push_back(active);
        ++particle_count_;
    }
    
    // Cache-friendly update - processes all particles in a single loop
    void update_positions(float dt) {
        for (std::size_t i = 0; i < particle_count_; ++i) {
            if (data_.active[i]) {
                data_.x[i] += data_.vx[i] * dt;
                data_.y[i] += data_.vy[i] * dt;
                data_.z[i] += data_.vz[i] * dt;
            }
        }
    }
    
    // SIMD-friendly operations
    void apply_gravity(float gravity, float dt) {
        // Process in chunks for SIMD optimization
        const std::size_t simd_width = 8; // Assuming AVX2 with 256-bit registers
        std::size_t i = 0;
        
        for (; i + simd_width <= particle_count_; i += simd_width) {
            // SIMD operations on 8 particles at once
            // This is pseudocode - actual implementation would use intrinsics
            for (std::size_t j = 0; j < simd_width; ++j) {
                if (data_.active[i + j]) {
                    data_.vy[i + j] -= gravity * dt;
                }
            }
        }
        
        // Handle remaining particles
        for (; i < particle_count_; ++i) {
            if (data_.active[i]) {
                data_.vy[i] -= gravity * dt;
            }
        }
    }
};
```

### Memory Alignment and Padding

```cpp
// Memory alignment optimization
template<typename T, std::size_t Alignment = alignof(T)>
class AlignedAllocator {
public:
    using value_type = T;
    
    T* allocate(std::size_t n) {
        if (n == 0) return nullptr;
        
        void* ptr = std::aligned_alloc(Alignment, n * sizeof(T));
        if (!ptr) throw std::bad_alloc();
        
        return static_cast<T*>(ptr);
    }
    
    void deallocate(T* ptr, std::size_t) {
        std::free(ptr);
    }
};

// Optimized structure layout
struct OptimizedStruct {
    // Group members by size to minimize padding
    double large1;      // 8 bytes
    double large2;      // 8 bytes
    int medium1;        // 4 bytes
    int medium2;        // 4 bytes
    short small1;       // 2 bytes
    short small2;       // 2 bytes
    char tiny1;         // 1 byte
    char tiny2;         // 1 byte
    // Total: 24 bytes with minimal padding
    
    // Compare with poor layout:
    // char tiny1;      // 1 byte + 7 bytes padding
    // double large1;   // 8 bytes
    // char tiny2;      // 1 byte + 7 bytes padding
    // double large2;   // 8 bytes
    // Total: 32 bytes with significant padding
};

// Using [[no_unique_address]] (C++20) to eliminate empty base class overhead
template<typename T, typename Allocator = std::allocator<T>>
class ContainerWithNoUniqueAddress {
private:
    [[no_unique_address]] Allocator alloc_;  // No space overhead if empty
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    
public:
    // Constructor and methods...
};
```

## Memory Management Patterns

### Object Lifetime Management

```cpp
// Arena allocator for temporary objects with same lifetime
class ArenaAllocator {
private:
    struct Block {
        char* memory;
        std::size_t size;
        std::size_t offset;
        Block* next;
        
        Block(std::size_t sz) : size(sz), offset(0), next(nullptr) {
            memory = static_cast<char*>(std::malloc(size));
            if (!memory) throw std::bad_alloc();
        }
        
        ~Block() {
            std::free(memory);
        }
    };
    
    Block* current_block_;
    std::size_t default_block_size_;
    
    void allocate_new_block(std::size_t min_size) {
        std::size_t block_size = std::max(default_block_size_, min_size);
        Block* new_block = new Block(block_size);
        new_block->next = current_block_;
        current_block_ = new_block;
    }
    
public:
    explicit ArenaAllocator(std::size_t default_block_size = 4096)
        : current_block_(nullptr), default_block_size_(default_block_size) {}
    
    ~ArenaAllocator() {
        while (current_block_) {
            Block* next = current_block_->next;
            delete current_block_;
            current_block_ = next;
        }
    }
    
    void* allocate(std::size_t size, std::size_t alignment = alignof(std::max_align_t)) {
        if (size == 0) return nullptr;
        
        // Align the offset
        std::size_t aligned_offset = (current_block_->offset + alignment - 1) & ~(alignment - 1);
        
        if (aligned_offset + size > current_block_->size) {
            // Need a new block
            allocate_new_block(size + alignment - 1);
            aligned_offset = 0;
        }
        
        void* ptr = current_block_->memory + aligned_offset;
        current_block_->offset = aligned_offset + size;
        return ptr;
    }
    
    template<typename T, typename... Args>
    T* construct(Args&&... args) {
        void* ptr = allocate(sizeof(T), alignof(T));
        return new(ptr) T(std::forward<Args>(args)...);
    }
    
    void reset() {
        // Reset all blocks to empty - objects not destructed
        for (Block* block = current_block_; block; block = block->next) {
            block->offset = 0;
        }
    }
    
    // Note: No individual deallocation - entire arena is reset at once
};

// Usage example
void performance_critical_function() {
    ArenaAllocator arena(8192);
    
    // Allocate many temporary objects
    for (int i = 0; i < 1000; ++i) {
        auto* obj = arena.construct<SomeClass>(/* constructor args */);
        // Use obj...
    }
    
    // All objects automatically "deallocated" when arena goes out of scope
}
```

### Memory-Efficient Containers

```cpp
// Small Vector Optimization (SVO) - similar to std::vector but with stack storage
template<typename T, std::size_t StackCapacity = 8>
class SmallVector {
private:
    alignas(T) char stack_storage_[sizeof(T) * StackCapacity];
    T* heap_storage_;
    std::size_t size_;
    std::size_t capacity_;
    
    T* data_ptr() {
        return (capacity_ <= StackCapacity) ? 
               reinterpret_cast<T*>(stack_storage_) : heap_storage_;
    }
    
    const T* data_ptr() const {
        return (capacity_ <= StackCapacity) ? 
               reinterpret_cast<const T*>(stack_storage_) : heap_storage_;
    }
    
public:
    SmallVector() : heap_storage_(nullptr), size_(0), capacity_(StackCapacity) {}
    
    ~SmallVector() {
        // Destruct elements
        for (std::size_t i = 0; i < size_; ++i) {
            data_ptr()[i].~T();
        }
        
        // Deallocate heap storage if used
        if (capacity_ > StackCapacity) {
            std::free(heap_storage_);
        }
    }
    
    void push_back(const T& value) {
        if (size_ == capacity_) {
            // Need to grow
            std::size_t new_capacity = capacity_ * 2;
            T* new_storage = static_cast<T*>(std::malloc(new_capacity * sizeof(T)));
            if (!new_storage) throw std::bad_alloc();
            
            // Move existing elements
            T* old_data = data_ptr();
            for (std::size_t i = 0; i < size_; ++i) {
                new(new_storage + i) T(std::move(old_data[i]));
                old_data[i].~T();
            }
            
            // Update pointers
            if (capacity_ > StackCapacity) {
                std::free(heap_storage_);
            }
            
            heap_storage_ = new_storage;
            capacity_ = new_capacity;
        }
        
        new(data_ptr() + size_) T(value);
        ++size_;
    }
    
    std::size_t size() const { return size_; }
    const T& operator[](std::size_t index) const { return data_ptr()[index]; }
    T& operator[](std::size_t index) { return data_ptr()[index]; }
};
```

## Memory Debugging and Profiling

### Memory Tracking Allocator

```cpp
// Allocator that tracks memory usage for debugging
template<typename T>
class TrackingAllocator {
private:
    static std::atomic<std::size_t> total_allocated_;
    static std::atomic<std::size_t> total_deallocated_;
    static std::atomic<std::size_t> current_usage_;
    static std::mutex tracking_mutex_;
    
public:
    using value_type = T;
    
    T* allocate(std::size_t n) {
        std::size_t size = n * sizeof(T);
        T* ptr = static_cast<T*>(std::malloc(size));
        if (!ptr) throw std::bad_alloc();
        
        {
            std::lock_guard<std::mutex> lock(tracking_mutex_);
            total_allocated_ += size;
            current_usage_ += size;
        }
        
        return ptr;
    }
    
    void deallocate(T* ptr, std::size_t n) {
        std::size_t size = n * sizeof(T);
        
        {
            std::lock_guard<std::mutex> lock(tracking_mutex_);
            total_deallocated_ += size;
            current_usage_ -= size;
        }
        
        std::free(ptr);
    }
    
    static std::size_t get_total_allocated() {
        return total_allocated_.load();
    }
    
    static std::size_t get_total_deallocated() {
        return total_deallocated_.load();
    }
    
    static std::size_t get_current_usage() {
        return current_usage_.load();
    }
    
    static void print_memory_stats() {
        std::lock_guard<std::mutex> lock(tracking_mutex_);
        std::cout << "Memory Stats:\n"
                  << "  Total Allocated: " << total_allocated_.load() << " bytes\n"
                  << "  Total Deallocated: " << total_deallocated_.load() << " bytes\n"
                  << "  Current Usage: " << current_usage_.load() << " bytes\n"
                  << "  Memory Leaks: " << (total_allocated_.load() - total_deallocated_.load()) << " bytes\n";
    }
};

template<typename T>
std::atomic<std::size_t> TrackingAllocator<T>::total_allocated_{0};

template<typename T>
std::atomic<std::size_t> TrackingAllocator<T>::total_deallocated_{0};

template<typename T>
std::atomic<std::size_t> TrackingAllocator<T>::current_usage_{0};

template<typename T>
std::mutex TrackingAllocator<T>::tracking_mutex_;
```

### Memory Leak Detection

```cpp
// Simple memory leak detector
class MemoryLeakDetector {
private:
    struct AllocationInfo {
        void* ptr;
        std::size_t size;
        std::string file;
        int line;
        std::chrono::steady_clock::time_point timestamp;
    };
    
    static std::unordered_map<void*, AllocationInfo> allocations_;
    static std::mutex detector_mutex_;
    
public:
    static void record_allocation(void* ptr, std::size_t size, 
                                 const char* file, int line) {
        std::lock_guard<std::mutex> lock(detector_mutex_);
        allocations_[ptr] = {ptr, size, file, line, std::chrono::steady_clock::now()};
    }
    
    static void record_deallocation(void* ptr) {
        std::lock_guard<std::mutex> lock(detector_mutex_);
        allocations_.erase(ptr);
    }
    
    static void report_leaks() {
        std::lock_guard<std::mutex> lock(detector_mutex_);
        if (allocations_.empty()) {
            std::cout << "No memory leaks detected.\n";
            return;
        }
        
        std::cout << "Memory leaks detected:\n";
        for (const auto& [ptr, info] : allocations_) {
            std::cout << "  Leak at " << ptr << " (" << info.size << " bytes) "
                      << "allocated in " << info.file << ":" << info.line << "\n";
        }
    }
};

std::unordered_map<void*, MemoryLeakDetector::AllocationInfo> MemoryLeakDetector::allocations_;
std::mutex MemoryLeakDetector::detector_mutex_;

// Custom new/delete operators for leak detection
void* operator new(std::size_t size, const char* file, int line) {
    void* ptr = std::malloc(size);
    if (!ptr) throw std::bad_alloc();
    MemoryLeakDetector::record_allocation(ptr, size, file, line);
    return ptr;
}

void operator delete(void* ptr) noexcept {
    MemoryLeakDetector::record_deallocation(ptr);
    std::free(ptr);
}

// Macro for easy usage
#define NEW new(__FILE__, __LINE__)

// Usage:
// auto* obj = NEW MyClass();  // Will be tracked for leaks
```

## Performance Considerations

### Memory Access Patterns

```cpp
// Demonstrating cache-friendly vs cache-unfriendly access patterns
class CachePerformanceDemo {
public:
    // Cache-friendly: sequential access
    static void sequential_access(std::vector<int>& data) {
        for (std::size_t i = 0; i < data.size(); ++i) {
            data[i] *= 2;  // Sequential memory access
        }
    }
    
    // Cache-unfriendly: strided access
    static void strided_access(std::vector<int>& data, std::size_t stride) {
        for (std::size_t i = 0; i < data.size(); i += stride) {
            data[i] *= 2;  // Non-sequential memory access
        }
    }
    
    // Cache-friendly: blocked/tiled access for 2D arrays
    static void tiled_matrix_operation(std::vector<std::vector<int>>& matrix, 
                                      std::size_t tile_size) {
        std::size_t rows = matrix.size();
        std::size_t cols = matrix[0].size();
        
        for (std::size_t row_tile = 0; row_tile < rows; row_tile += tile_size) {
            for (std::size_t col_tile = 0; col_tile < cols; col_tile += tile_size) {
                // Process tile
                for (std::size_t i = row_tile; i < std::min(row_tile + tile_size, rows); ++i) {
                    for (std::size_t j = col_tile; j < std::min(col_tile + tile_size, cols); ++j) {
                        matrix[i][j] *= 2;
                    }
                }
            }
        }
    }
};
```

### Memory Prefetching

```cpp
// Manual memory prefetching for performance optimization
class PrefetchingDemo {
public:
    // Prefetching for linear access patterns
    static void prefetching_linear_access(std::vector<int>& data) {
        const std::size_t prefetch_distance = 64; // Number of elements to prefetch ahead
        
        for (std::size_t i = 0; i < data.size(); ++i) {
            // Prefetch data ahead
            if (i + prefetch_distance < data.size()) {
                __builtin_prefetch(&data[i + prefetch_distance], 0, 3);
            }
            
            // Process current element
            data[i] *= 2;
        }
    }
    
    // Prefetching for tree traversal
    struct TreeNode {
        int value;
        TreeNode* left;
        TreeNode* right;
        
        TreeNode(int val) : value(val), left(nullptr), right(nullptr) {}
    };
    
    static void prefetching_tree_traversal(TreeNode* root) {
        if (!root) return;
        
        // Prefetch children
        __builtin_prefetch(root->left, 0, 3);
        __builtin_prefetch(root->right, 0, 3);
        
        // Process current node
        std::cout << root->value << " ";
        
        // Traverse children
        prefetching_tree_traversal(root->left);
        prefetching_tree_traversal(root->right);
    }
};
```

## Best Practices

### Memory Management Guidelines

1. **Choose the Right Allocation Strategy** - Stack for small, short-lived objects; heap for large or long-lived objects
2. **Minimize Allocations** - Reuse memory when possible and pre-allocate when sizes are known
3. **Consider Alignment** - Proper alignment can significantly impact performance
4. **Use RAII** - Automatic resource management prevents leaks and ensures proper cleanup
5. **Profile Memory Usage** - Regular profiling helps identify bottlenecks and leaks
6. **Understand Cache Behavior** - Design data structures for cache-friendly access patterns
7. **Use Standard Allocators** - Leverage std::pmr and other standard library allocators
8. **Test Multi-threaded Scenarios** - Ensure thread safety in concurrent environments

Advanced memory management is crucial for developing high-performance C++ applications and understanding how modern libraries efficiently manage resources.