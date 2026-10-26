# Memory Management Strategies

## Introduction

Memory management is a critical aspect of C++ programming that directly impacts application performance, resource utilization, and system stability. Understanding various memory management strategies is essential for developing efficient C++ applications and implementing high-performance libraries.

## Memory Hierarchy and Characteristics

### Memory Types

1. **Stack Memory**
   - Fast allocation/deallocation (single CPU instruction)
   - Limited size (typically 1-8 MB)
   - Automatic lifetime management
   - LIFO allocation pattern

2. **Heap Memory**
   - Dynamic allocation
   - Larger size (limited by system memory)
   - Manual lifetime management
   - Potentially fragmented

3. **Static Memory**
   - Compile-time allocation
   - Fixed size
   - Global lifetime

### Memory Access Patterns

```cpp
// Stack allocation - fastest
void stack_example() {
    int array[1000]; // Allocated on stack
    // Automatically deallocated when function exits
}

// Heap allocation - more flexible but slower
void heap_example() {
    auto array = std::make_unique<int[]>(1000); // Allocated on heap
    // Manually managed or automatically via RAII
}
```

## Stack Memory Management

### Stack Allocation Patterns

```cpp
class StackAllocator {
private:
    void* start_;
    std::size_t size_;
    std::size_t offset_;

public:
    StackAllocator(void* start, std::size_t size)
        : start_(start), size_(size), offset_(0) {}

    template<typename T>
    T* allocate(std::size_t count = 1) {
        const std::size_t alignment = alignof(T);
        const std::size_t allocated_size = count * sizeof(T);
        
        // Align the offset
        std::size_t aligned_offset = (offset_ + alignment - 1) & ~(alignment - 1);
        
        if (aligned_offset + allocated_size > size_) {
            throw std::bad_alloc();
        }
        
        T* ptr = reinterpret_cast<T*>(static_cast<char*>(start_) + aligned_offset);
        offset_ = aligned_offset + allocated_size;
        return ptr;
    }
    
    void reset() {
        offset_ = 0;
    }
    
    std::size_t used() const {
        return offset_;
    }
};

// Usage example
alignas(std::max_align_t) char buffer[4096];
StackAllocator allocator(buffer, sizeof(buffer));

int* int_array = allocator.allocate<int>(100);
float* float_array = allocator.allocate<float>(50);
// Reset when done with temporary allocations
allocator.reset();
```

### Stack Unwinding

```cpp
// Exception-safe stack allocation
template<typename T>
class StackObject {
    T* ptr_;
    StackAllocator& allocator_;
    
public:
    StackObject(StackAllocator& allocator) 
        : allocator_(allocator) {
        ptr_ = allocator_.template allocate<T>();
        new(ptr_) T(); // Construct object
    }
    
    ~StackObject() {
        if (ptr_) {
            ptr_->~T(); // Destruct object
        }
    }
    
    T* operator->() { return ptr_; }
    T& operator*() { return *ptr_; }
};
```

## Heap Memory Management

### Memory Pool Pattern

```cpp
template<typename T, std::size_t BlockSize = 4096>
class MemoryPool {
private:
    struct Block {
        alignas(T) char data[sizeof(T)];
        Block* next;
    };
    
    struct Pool {
        char* memory_pool;
        Block* free_list;
        std::size_t pool_size;
        Pool* next_pool;
        
        Pool(std::size_t size) : pool_size(size), next_pool(nullptr) {
            memory_pool = static_cast<char*>(std::malloc(size));
            if (!memory_pool) throw std::bad_alloc();
            
            // Initialize free list
            const std::size_t block_count = size / sizeof(Block);
            free_list = reinterpret_cast<Block*>(memory_pool);
            
            Block* current = free_list;
            for (std::size_t i = 0; i < block_count - 1; ++i) {
                current->next = reinterpret_cast<Block*>(
                    reinterpret_cast<char*>(current) + sizeof(Block));
                current = current->next;
            }
            current->next = nullptr;
        }
        
        ~Pool() {
            std::free(memory_pool);
        }
    };
    
    Pool* current_pool_;
    std::mutex pool_mutex_;

public:
    MemoryPool() : current_pool_(nullptr) {}
    
    ~MemoryPool() {
        while (current_pool_) {
            Pool* next = current_pool_->next_pool;
            delete current_pool_;
            current_pool_ = next;
        }
    }
    
    T* allocate() {
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        if (!current_pool_ || !current_pool_->free_list) {
            // Create new pool
            Pool* new_pool = new Pool(BlockSize);
            new_pool->next_pool = current_pool_;
            current_pool_ = new_pool;
        }
        
        Block* block = current_pool_->free_list;
        current_pool_->free_list = block->next;
        return reinterpret_cast<T*>(block);
    }
    
    void deallocate(T* ptr) {
        if (!ptr) return;
        
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        Block* block = reinterpret_cast<Block*>(ptr);
        block->next = current_pool_->free_list;
        current_pool_->free_list = block;
    }
};
```

### Buddy Allocator

```cpp
class BuddyAllocator {
private:
    struct Block {
        bool free;
        std::size_t order; // Size = 2^order
        Block* next;
        Block* prev;
    };
    
    std::vector<Block*> free_lists_; // One list per order
    void* memory_start_;
    std::size_t total_size_;
    std::size_t min_order_;
    
    std::size_t get_order(std::size_t size) {
        std::size_t order = min_order_;
        while ((1ULL << order) < size) {
            ++order;
        }
        return order;
    }
    
    Block* get_buddy(Block* block) {
        std::size_t block_size = 1ULL << block->order;
        std::size_t offset = reinterpret_cast<char*>(block) - 
                            static_cast<char*>(memory_start_);
        std::size_t buddy_offset = offset ^ block_size;
        return reinterpret_cast<Block*>(
            static_cast<char*>(memory_start_) + buddy_offset);
    }
    
public:
    BuddyAllocator(void* memory, std::size_t size) 
        : memory_start_(memory), total_size_(size) {
        // Calculate minimum order (smallest block size)
        min_order_ = 0;
        while ((1ULL << min_order_) < sizeof(Block)) {
            ++min_order_;
        }
        
        // Initialize free lists
        std::size_t max_order = 0;
        while ((1ULL << max_order) < size) {
            ++max_order;
        }
        free_lists_.resize(max_order + 1, nullptr);
        
        // Create initial block
        Block* initial_block = static_cast<Block*>(memory_start_);
        initial_block->free = true;
        initial_block->order = max_order;
        initial_block->next = nullptr;
        initial_block->prev = nullptr;
        
        free_lists_[max_order] = initial_block;
    }
    
    void* allocate(std::size_t size) {
        std::size_t required_order = get_order(size + sizeof(Block));
        
        // Find suitable block
        std::size_t order = required_order;
        while (order < free_lists_.size() && !free_lists_[order]) {
            ++order;
        }
        
        if (order >= free_lists_.size()) {
            return nullptr; // No suitable block found
        }
        
        // Split blocks if necessary
        while (order > required_order) {
            Block* block = free_lists_[order];
            free_lists_[order] = block->next;
            
            --order;
            std::size_t block_size = 1ULL << order;
            
            Block* buddy = reinterpret_cast<Block*>(
                reinterpret_cast<char*>(block) + block_size);
            buddy->free = false;
            buddy->order = order;
            
            // Add buddy to free list
            buddy->next = free_lists_[order];
            buddy->prev = nullptr;
            if (free_lists_[order]) {
                free_lists_[order]->prev = buddy;
            }
            free_lists_[order] = buddy;
            
            block->order = order;
        }
        
        // Remove block from free list
        Block* block = free_lists_[order];
        free_lists_[order] = block->next;
        if (free_lists_[order]) {
            free_lists_[order]->prev = nullptr;
        }
        block->free = false;
        
        return reinterpret_cast<char*>(block) + sizeof(Block);
    }
    
    void deallocate(void* ptr) {
        if (!ptr) return;
        
        Block* block = reinterpret_cast<Block*>(
            static_cast<char*>(ptr) - sizeof(Block));
        block->free = true;
        
        // Try to coalesce with buddies
        std::size_t order = block->order;
        while (order < free_lists_.size() - 1) {
            Block* buddy = get_buddy(block);
            if (!buddy->free || buddy->order != order) {
                break;
            }
            
            // Remove buddy from free list
            if (buddy->prev) {
                buddy->prev->next = buddy->next;
            } else {
                free_lists_[order] = buddy->next;
            }
            if (buddy->next) {
                buddy->next->prev = buddy->prev;
            }
            
            // Coalesce blocks
            if (buddy < block) {
                block = buddy;
            }
            ++order;
            block->order = order;
        }
        
        // Add block to free list
        block->next = free_lists_[order];
        block->prev = nullptr;
        if (free_lists_[order]) {
            free_lists_[order]->prev = block;
        }
        free_lists_[order] = block;
    }
};
```

## Memory Management Patterns

### Object Pool Pattern

```cpp
template<typename T>
class ObjectPool {
private:
    struct Node {
        alignas(T) char data[sizeof(T)];
        Node* next;
    };
    
    Node* free_list_;
    std::vector<std::unique_ptr<Node[]>> chunks_;
    std::mutex mutex_;
    
    void allocate_chunk(std::size_t chunk_size) {
        auto chunk = std::make_unique<Node[]>(chunk_size);
        for (std::size_t i = 0; i < chunk_size - 1; ++i) {
            chunk[i].next = &chunk[i + 1];
        }
        chunk[chunk_size - 1].next = free_list_;
        free_list_ = chunk.get();
        chunks_.push_back(std::move(chunk));
    }
    
public:
    ObjectPool(std::size_t initial_chunk_size = 32) 
        : free_list_(nullptr) {
        allocate_chunk(initial_chunk_size);
    }
    
    template<typename... Args>
    T* construct(Args&&... args) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        if (!free_list_) {
            allocate_chunk(32); // Allocate new chunk if needed
        }
        
        Node* node = free_list_;
        free_list_ = node->next;
        
        T* object = reinterpret_cast<T*>(node->data);
        new(object) T(std::forward<Args>(args)...);
        return object;
    }
    
    void destroy(T* object) {
        if (!object) return;
        
        object->~T();
        
        std::lock_guard<std::mutex> lock(mutex_);
        Node* node = reinterpret_cast<Node*>(
            reinterpret_cast<char*>(object) - offsetof(Node, data));
        node->next = free_list_;
        free_list_ = node;
    }
};
```

### Slab Allocator

```cpp
template<typename T>
class SlabAllocator {
private:
    struct Slab {
        alignas(T) char data[sizeof(T)];
    };
    
    struct SlabPage {
        static constexpr std::size_t PAGE_SIZE = 4096;
        static constexpr std::size_t SLABS_PER_PAGE = 
            (PAGE_SIZE - sizeof(SlabPage*)) / sizeof(Slab);
            
        Slab slabs[SLABS_PER_PAGE];
        bool allocated[SLABS_PER_PAGE];
        SlabPage* next;
        
        SlabPage() : next(nullptr) {
            std::fill(std::begin(allocated), std::end(allocated), false);
        }
    };
    
    SlabPage* current_page_;
    std::vector<std::unique_ptr<SlabPage>> pages_;
    std::mutex mutex_;
    
    Slab* find_free_slab() {
        if (!current_page_) {
            pages_.push_back(std::make_unique<SlabPage>());
            current_page_ = pages_.back().get();
        }
        
        for (std::size_t i = 0; i < SlabPage::SLABS_PER_PAGE; ++i) {
            if (!current_page_->allocated[i]) {
                current_page_->allocated[i] = true;
                return &current_page_->slabs[i];
            }
        }
        
        // Current page is full, allocate new page
        pages_.push_back(std::make_unique<SlabPage>());
        pages_.back()->next = current_page_;
        current_page_ = pages_.back().get();
        current_page_->allocated[0] = true;
        return &current_page_->slabs[0];
    }
    
public:
    SlabAllocator() : current_page_(nullptr) {}
    
    template<typename... Args>
    T* construct(Args&&... args) {
        std::lock_guard<std::mutex> lock(mutex_);
        Slab* slab = find_free_slab();
        T* object = reinterpret_cast<T*>(slab->data);
        new(object) T(std::forward<Args>(args)...);
        return object;
    }
    
    void destroy(T* object) {
        if (!object) return;
        
        object->~T();
        
        std::lock_guard<std::mutex> lock(mutex_);
        // In a real implementation, we would track which slab this object belongs to
        // and mark it as free, but for simplicity we'll just leave it allocated
    }
};
```

## Memory Management in STL

### std::vector Memory Strategy

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
private:
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    Allocator alloc_;
    
    // Exponential growth strategy
    void reallocate(std::size_t new_capacity) {
        T* new_data = alloc_.allocate(new_capacity);
        
        // Move existing elements
        for (std::size_t i = 0; i < size_; ++i) {
            alloc_.construct(new_data + i, std::move(data_[i]));
            alloc_.destroy(data_ + i);
        }
        
        alloc_.deallocate(data_, capacity_);
        data_ = new_data;
        capacity_ = new_capacity;
    }
    
public:
    void push_back(const T& value) {
        if (size_ == capacity_) {
            // Exponential growth: typically 1.5x or 2x
            std::size_t new_capacity = capacity_ == 0 ? 1 : capacity_ * 2;
            reallocate(new_capacity);
        }
        
        alloc_.construct(data_ + size_, value);
        ++size_;
    }
    
    void push_back(T&& value) {
        if (size_ == capacity_) {
            std::size_t new_capacity = capacity_ == 0 ? 1 : capacity_ * 2;
            reallocate(new_capacity);
        }
        
        alloc_.construct(data_ + size_, std::move(value));
        ++size_;
    }
};
```

### std::deque Memory Strategy

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class deque {
private:
    static constexpr std::size_t BLOCK_SIZE = 512 / sizeof(T);
    
    struct Block {
        alignas(T) char data[BLOCK_SIZE * sizeof(T)];
    };
    
    std::vector<Block*> map_; // Pointers to blocks
    std::size_t start_index_; // Index of first element
    std::size_t end_index_;   // Index one past last element
    Allocator alloc_;
    
public:
    void push_back(const T& value) {
        std::size_t block_index = end_index_ / BLOCK_SIZE;
        std::size_t element_index = end_index_ % BLOCK_SIZE;
        
        if (block_index >= map_.size()) {
            // Allocate new block
            map_.push_back(alloc_.allocate(1));
        }
        
        T* element = reinterpret_cast<T*>(
            map_[block_index]->data + element_index * sizeof(T));
        new(element) T(value);
        ++end_index_;
    }
};
```

## Performance Considerations

### Memory Fragmentation

```cpp
// Memory defragmentation strategy
class FragmentationAwareAllocator {
private:
    struct Block {
        void* ptr;
        std::size_t size;
        bool allocated;
        Block* next;
    };
    
    Block* free_blocks_;
    Block* allocated_blocks_;
    
public:
    void* allocate(std::size_t size) {
        // Try to find a suitable free block first
        Block** prev = &free_blocks_;
        for (Block* current = free_blocks_; current; 
             prev = &current->next, current = current->next) {
            if (current->size >= size && !current->allocated) {
                current->allocated = true;
                // Move to allocated list
                *prev = current->next;
                current->next = allocated_blocks_;
                allocated_blocks_ = current;
                return current->ptr;
            }
        }
        
        // No suitable block found, allocate new memory
        void* ptr = std::malloc(size);
        if (!ptr) return nullptr;
        
        Block* new_block = new Block{ptr, size, true, allocated_blocks_};
        allocated_blocks_ = new_block;
        return ptr;
    }
    
    void deallocate(void* ptr) {
        // Find the block
        Block** prev = &allocated_blocks_;
        for (Block* current = allocated_blocks_; current; 
             prev = &current->next, current = current->next) {
            if (current->ptr == ptr) {
                // Move to free list
                *prev = current->next;
                current->next = free_blocks_;
                current->allocated = false;
                free_blocks_ = current;
                return;
            }
        }
    }
    
    void defragment() {
        // Simple defragmentation: consolidate adjacent free blocks
        // This is a simplified version - real implementation would be more complex
    }
};
```

### Cache-Friendly Allocation

```cpp
// Allocator that tries to keep related objects close together
template<typename T>
class CacheFriendlyAllocator {
private:
    static constexpr std::size_t CACHE_LINE_SIZE = 64;
    
    struct CacheLine {
        alignas(CACHE_LINE_SIZE) char data[CACHE_LINE_SIZE];
    };
    
    struct Block {
        CacheLine* lines;
        std::size_t line_count;
        std::vector<bool> allocated; // Track which cache lines are used
    };
    
    std::vector<std::unique_ptr<Block>> blocks_;
    
public:
    T* allocate() {
        // Try to find space in existing blocks
        for (auto& block : blocks_) {
            std::size_t required_lines = 
                (sizeof(T) + CACHE_LINE_SIZE - 1) / CACHE_LINE_SIZE;
                
            // Find consecutive free cache lines
            for (std::size_t i = 0; i <= block->line_count - required_lines; ++i) {
                bool available = true;
                for (std::size_t j = 0; j < required_lines; ++j) {
                    if (block->allocated[i + j]) {
                        available = false;
                        break;
                    }
                }
                
                if (available) {
                    // Mark lines as allocated
                    for (std::size_t j = 0; j < required_lines; ++j) {
                        block->allocated[i + j] = true;
                    }
                    
                    return reinterpret_cast<T*>(
                        block->lines[i].data);
                }
            }
        }
        
        // Create new block
        constexpr std::size_t LINES_PER_BLOCK = 64;
        auto block = std::make_unique<Block>();
        block->lines = static_cast<CacheLine*>(
            std::aligned_alloc(CACHE_LINE_SIZE, 
                              sizeof(CacheLine) * LINES_PER_BLOCK));
        block->line_count = LINES_PER_BLOCK;
        block->allocated.resize(LINES_PER_BLOCK, false);
        
        // Allocate first lines for the object
        std::size_t required_lines = 
            (sizeof(T) + CACHE_LINE_SIZE - 1) / CACHE_LINE_SIZE;
        for (std::size_t i = 0; i < required_lines; ++i) {
            block->allocated[i] = true;
        }
        
        T* result = reinterpret_cast<T*>(block->lines[0].data);
        blocks_.push_back(std::move(block));
        return result;
    }
};
```

## Best Practices

### Memory Management Guidelines

1. **RAII Principle** - Always use RAII for automatic resource management
2. **Minimize Allocations** - Reduce the number of allocations when possible
3. **Prefer Stack Allocation** - Use stack allocation for small, short-lived objects
4. **Custom Allocators** - Use custom allocators for specific performance requirements
5. **Profile Memory Usage** - Regularly profile memory usage to identify bottlenecks
6. **Avoid Memory Leaks** - Use smart pointers and RAII consistently
7. **Consider Alignment** - Always respect alignment requirements
8. **Thread Safety** - Ensure thread safety in multi-threaded environments

Understanding these memory management strategies is crucial for developing high-performance C++ applications and for comprehending how the STL manages memory internally.