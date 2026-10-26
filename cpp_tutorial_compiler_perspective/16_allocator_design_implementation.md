# Allocator Design and Implementation

## Introduction

Allocators in C++ are objects responsible for managing memory allocation and deallocation for containers. They are a fundamental component of the Standard Template Library (STL), allowing containers to be decoupled from specific memory management strategies. Understanding allocator design is crucial for implementing custom containers and optimizing memory usage.

## Allocator Requirements

### Basic Allocator Interface

A minimal allocator must provide the following:

```cpp
template<typename T>
class allocator {
public:
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;

    // Constructors
    allocator() = default;
    template<typename U>
    allocator(const allocator<U>&) noexcept {}

    // Allocation functions
    T* allocate(std::size_t n) {
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T))
            throw std::bad_alloc();
        
        void* ptr = std::malloc(n * sizeof(T));
        if (!ptr)
            throw std::bad_alloc();
            
        return static_cast<T*>(ptr);
    }

    void deallocate(T* ptr, std::size_t) {
        std::free(ptr);
    }

    // Construction and destruction
    template<typename U, typename... Args>
    void construct(U* ptr, Args&&... args) {
        ::new(ptr) U(std::forward<Args>(args)...);
    }

    template<typename U>
    void destroy(U* ptr) {
        ptr->~U();
    }

    // Comparison
    template<typename U>
    bool operator==(const allocator<U>&) const noexcept {
        return true;
    }

    template<typename U>
    bool operator!=(const allocator<U>&) const noexcept {
        return false;
    }
};
```

## Allocator Concepts

### Allocator Requirements (C++17)

An allocator must satisfy the Allocator requirements:
1. **Copy constructible** - Can be copied
2. **Equality comparable** - Can be compared for equality
3. **Provide allocate/deallocate** - Memory management functions
4. **Provide construct/destroy** - Object construction/destruction

### Allocator Types

```cpp
// Stateful allocator (has internal state)
class stateful_allocator {
    std::pmr::memory_resource* resource;
public:
    // Different instances may behave differently
};

// Stateless allocator (no internal state)
template<typename T>
class stateless_allocator {
    // All instances are equivalent
};
```

## Custom Allocator Implementation

### Pool Allocator

```cpp
template<typename T, std::size_t BlockSize = 4096>
class pool_allocator {
private:
    struct block {
        alignas(T) char data[sizeof(T)];
        block* next;
    };

    struct pool {
        block* free_list;
        char* memory_pool;
        std::size_t pool_size;
        pool* next_pool;
        
        pool(std::size_t size) : pool_size(size), next_pool(nullptr) {
            memory_pool = static_cast<char*>(std::malloc(size));
            if (!memory_pool)
                throw std::bad_alloc();
                
            // Initialize free list
            free_list = reinterpret_cast<block*>(memory_pool);
            block* current = free_list;
            
            for (std::size_t i = 0; i < (size / sizeof(block)) - 1; ++i) {
                current->next = reinterpret_cast<block*>(
                    reinterpret_cast<char*>(current) + sizeof(block));
                current = current->next;
            }
            current->next = nullptr;
        }
        
        ~pool() {
            std::free(memory_pool);
        }
    };

    pool* current_pool;
    std::mutex pool_mutex; // For thread safety

public:
    using value_type = T;
    
    pool_allocator() : current_pool(nullptr) {}
    
    template<typename U>
    pool_allocator(const pool_allocator<U, BlockSize>& other) noexcept
        : current_pool(other.current_pool) {}
    
    T* allocate(std::size_t n) {
        if (n != 1)
            return static_cast<T*>(std::malloc(n * sizeof(T)));
            
        std::lock_guard<std::mutex> lock(pool_mutex);
        
        if (!current_pool || !current_pool->free_list) {
            // Create new pool
            pool* new_pool = new pool(BlockSize);
            new_pool->next_pool = current_pool;
            current_pool = new_pool;
        }
        
        block* result = current_pool->free_list;
        current_pool->free_list = result->next;
        return reinterpret_cast<T*>(result);
    }
    
    void deallocate(T* ptr, std::size_t n) {
        if (n != 1) {
            std::free(ptr);
            return;
        }
        
        std::lock_guard<std::mutex> lock(pool_mutex);
        
        block* returned_block = reinterpret_cast<block*>(ptr);
        returned_block->next = current_pool->free_list;
        current_pool->free_list = returned_block;
    }
    
    template<typename U>
    bool operator==(const pool_allocator<U, BlockSize>&) const noexcept {
        return true;
    }
    
    template<typename U>
    bool operator!=(const pool_allocator<U, BlockSize>&) const noexcept {
        return false;
    }
};
```

## Allocator-Aware Containers

### Container Requirements

Containers must properly handle allocators:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
private:
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    Allocator alloc_;

public:
    using allocator_type = Allocator;
    using value_type = T;
    
    // Allocator-aware constructors
    explicit vector(const Allocator& alloc = Allocator())
        : data_(nullptr), size_(0), capacity_(0), alloc_(alloc) {}
        
    vector(std::size_t count, const T& value, 
           const Allocator& alloc = Allocator())
        : size_(count), capacity_(count), alloc_(alloc) {
        data_ = alloc_.allocate(count);
        for (std::size_t i = 0; i < count; ++i)
            alloc_.construct(data_ + i, value);
    }
    
    // Copy constructor with allocator
    vector(const vector& other, const Allocator& alloc)
        : size_(other.size_), capacity_(other.capacity_), alloc_(alloc) {
        data_ = alloc_.allocate(capacity_);
        for (std::size_t i = 0; i < size_; ++i)
            alloc_.construct(data_ + i, other.data_[i]);
    }
    
    // Move constructor with allocator
    vector(vector&& other, const Allocator& alloc)
        : data_(nullptr), size_(0), capacity_(0), alloc_(alloc) {
        if (alloc_ == other.alloc_) {
            // Same allocator, can transfer ownership
            data_ = other.data_;
            size_ = other.size_;
            capacity_ = other.capacity_;
            other.data_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        } else {
            // Different allocators, must move elements
            data_ = alloc_.allocate(other.capacity_);
            for (std::size_t i = 0; i < other.size_; ++i) {
                alloc_.construct(data_ + i, std::move(other.data_[i]));
            }
            size_ = other.size_;
            capacity_ = other.capacity_;
        }
    }
};
```

## Allocator Propagation

### Propagation Traits

```cpp
// Traits to control allocator propagation
template<typename Allocator>
struct allocator_traits {
    using allocator_type = Allocator;
    using value_type = typename Allocator::value_type;
    using pointer = typename Allocator::pointer;
    using const_pointer = typename Allocator::const_pointer;
    using void_pointer = typename Allocator::void_pointer;
    using const_void_pointer = typename Allocator::const_void_pointer;
    using difference_type = typename Allocator::difference_type;
    using size_type = typename Allocator::size_type;
    
    // Propagation control
    using propagate_on_container_copy_assignment = 
        typename Allocator::propagate_on_container_copy_assignment;
    using propagate_on_container_move_assignment = 
        typename Allocator::propagate_on_container_move_assignment;
    using propagate_on_container_swap = 
        typename Allocator::propagate_on_container_swap;
    using is_always_equal = 
        typename Allocator::is_always_equal;
};
```

### Container Operations with Propagation

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class container {
    using traits = std::allocator_traits<Allocator>;
    
public:
    // Copy assignment with allocator propagation
    container& operator=(const container& other) {
        if (this != &other) {
            if constexpr (traits::propagate_on_container_copy_assignment::value) {
                if (alloc_ != other.alloc_) {
                    // Destroy existing elements
                    clear();
                    // Deallocate memory
                    traits::deallocate(alloc_, data_, capacity_);
                    // Propagate allocator
                    alloc_ = other.alloc_;
                    // Allocate new memory
                    data_ = traits::allocate(alloc_, other.capacity_);
                    capacity_ = other.capacity_;
                }
            }
            // Copy elements
            assign_elements(other);
        }
        return *this;
    }
    
    // Move assignment with allocator propagation
    container& operator=(container&& other) noexcept(
        traits::propagate_on_container_move_assignment::value ||
        traits::is_always_equal::value) {
        if (this != &other) {
            if constexpr (traits::propagate_on_container_move_assignment::value) {
                // Destroy existing elements
                clear();
                // Deallocate memory
                traits::deallocate(alloc_, data_, capacity_);
                // Propagate allocator
                alloc_ = std::move(other.alloc_);
                // Transfer resources
                data_ = other.data_;
                size_ = other.size_;
                capacity_ = other.capacity_;
                // Reset other
                other.data_ = nullptr;
                other.size_ = 0;
                other.capacity_ = 0;
            } else if (alloc_ == other.alloc_) {
                // Same allocator, transfer resources
                clear();
                traits::deallocate(alloc_, data_, capacity_);
                data_ = other.data_;
                size_ = other.size_;
                capacity_ = other.capacity_;
                other.data_ = nullptr;
                other.size_ = 0;
                other.capacity_ = 0;
            } else {
                // Different allocators, move elements
                assign_elements(std::move(other));
            }
        }
        return *this;
    }
};
```

## STL Allocator Implementation

### std::allocator

The standard allocator is a simple wrapper around `malloc`/`free`:

```cpp
template<typename T>
class allocator {
public:
    using value_type = T;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
    
    allocator() = default;
    
    template<typename U>
    allocator(const allocator<U>&) noexcept {}
    
    T* allocate(std::size_t n) {
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T))
            throw std::bad_alloc();
            
        void* ptr = std::malloc(n * sizeof(T));
        if (!ptr)
            throw std::bad_alloc();
            
        return static_cast<T*>(ptr);
    }
    
    void deallocate(T* ptr, std::size_t) {
        std::free(ptr);
    }
};
```

### Polymorphic Allocators (C++17)

```cpp
namespace pmr {
    class polymorphic_allocator {
        std::pmr::memory_resource* resource_;
        
    public:
        polymorphic_allocator() noexcept
            : resource_(std::pmr::get_default_resource()) {}
            
        polymorphic_allocator(std::pmr::memory_resource* r) noexcept
            : resource_(r) {}
            
        template<typename T>
        T* allocate(std::size_t n) {
            return static_cast<T*>(resource_->allocate(n * sizeof(T), alignof(T)));
        }
        
        template<typename T>
        void deallocate(T* ptr, std::size_t n) {
            resource_->deallocate(ptr, n * sizeof(T), alignof(T));
        }
    };
}
```

## Performance Considerations

### Memory Fragmentation

Custom allocators can reduce fragmentation:

```cpp
// Stack allocator for temporary allocations
template<std::size_t Size>
class stack_allocator {
    alignas(std::max_align_t) char buffer[Size];
    std::size_t offset;
    
public:
    stack_allocator() : offset(0) {}
    
    void* allocate(std::size_t size, std::size_t alignment) {
        std::size_t aligned_offset = (offset + alignment - 1) & ~(alignment - 1);
        if (aligned_offset + size > Size)
            throw std::bad_alloc();
            
        void* ptr = buffer + aligned_offset;
        offset = aligned_offset + size;
        return ptr;
    }
    
    void deallocate(void* ptr, std::size_t size) {
        // Stack-like deallocation - only works for LIFO order
        char* char_ptr = static_cast<char*>(ptr);
        if (char_ptr + size == buffer + offset) {
            offset = char_ptr - buffer;
        }
    }
};
```

### Thread Safety

```cpp
// Thread-safe allocator
template<typename T>
class thread_safe_allocator {
    std::mutex mutex_;
    std::allocator<T> alloc_;
    
public:
    using value_type = T;
    
    T* allocate(std::size_t n) {
        std::lock_guard<std::mutex> lock(mutex_);
        return alloc_.allocate(n);
    }
    
    void deallocate(T* ptr, std::size_t n) {
        std::lock_guard<std::mutex> lock(mutex_);
        alloc_.deallocate(ptr, n);
    }
};
```

## Best Practices

### Allocator Design Guidelines

1. **Stateless when possible** - Stateless allocators are simpler and more efficient
2. **Proper alignment** - Always respect alignment requirements
3. **Exception safety** - Handle allocation failures gracefully
4. **Thread safety** - Consider thread safety requirements
5. **Resource management** - Properly manage underlying resources

### Container Implementation

1. **Use allocator_traits** - Always use `std::allocator_traits` for allocator operations
2. **Propagate correctly** - Implement proper allocator propagation semantics
3. **Handle equality** - Implement correct allocator equality comparison
4. **Optimize for common cases** - Optimize for the most common allocator usage patterns

Understanding allocator design and implementation is essential for developing efficient C++ containers and libraries, and for mastering the inner workings of the STL.