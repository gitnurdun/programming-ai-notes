# Advanced Exception Safety

## Introduction

Advanced exception safety goes beyond basic exception handling to ensure that programs maintain their invariants and behave correctly even in the presence of exceptions. This is particularly important for developing robust libraries and understanding how the STL maintains its guarantees.

## Exception Safety Guarantees

### The Three Levels of Exception Safety

1. **No-throw guarantee** - Operations never throw exceptions
2. **Strong guarantee** - Operations succeed completely or have no effect
3. **Basic guarantee** - No resources are leaked, but object state may be modified

### Implementing Strong Exception Safety

```cpp
// Example: Strong exception safety in a container's push_back operation
template<typename T, typename Allocator = std::allocator<T>>
class StrongExceptionSafeVector {
private:
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    Allocator alloc_;
    
    void reallocate_strong(std::size_t new_capacity) {
        // Strong exception safety: if this fails, object remains unchanged
        
        // Create temporary container (can throw)
        StrongExceptionSafeVector temp;
        temp.data_ = std::allocator_traits<Allocator>::allocate(temp.alloc_, new_capacity);
        temp.capacity_ = new_capacity;
        
        // Guard to clean up if exception occurs
        struct Guard {
            StrongExceptionSafeVector& vec;
            bool active = true;
            
            ~Guard() {
                if (active) {
                    for (std::size_t i = 0; i < vec.size_; ++i) {
                        std::allocator_traits<Allocator>::destroy(vec.alloc_, vec.data_ + i);
                    }
                    std::allocator_traits<Allocator>::deallocate(vec.alloc_, vec.data_, vec.capacity_);
                }
            }
        } guard{*this};
        
        try {
            // Copy elements (can throw)
            for (std::size_t i = 0; i < size_; ++i) {
                std::allocator_traits<Allocator>::construct(temp.alloc_, temp.data_ + i, data_[i]);
                ++temp.size_;
            }
        } catch (...) {
            // Clean up temporary container
            for (std::size_t i = 0; i < temp.size_; ++i) {
                std::allocator_traits<Allocator>::destroy(temp.alloc_, temp.data_ + i);
            }
            std::allocator_traits<Allocator>::deallocate(temp.alloc_, temp.data_, temp.capacity_);
            throw; // Re-throw original exception
        }
        
        // No exceptions beyond this point
        // Clean up old data
        for (std::size_t i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        }
        std::allocator_traits<Allocator>::deallocate(alloc_, data_, capacity_);
        
        // Transfer ownership
        data_ = temp.data_;
        capacity_ = temp.capacity_;
        size_ = temp.size_;
        
        // Deactivate guard
        temp.data_ = nullptr;
        temp.capacity_ = 0;
        temp.size_ = 0;
        guard.active = false;
    }
    
public:
    void push_back(const T& value) {
        if (size_ == capacity_) {
            reallocate_strong(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        
        // This operation provides strong exception safety
        // If it throws, object state remains unchanged
        std::allocator_traits<Allocator>::construct(alloc_, data_ + size_, value);
        ++size_;
    }
};
```

## Exception-Safe Resource Management

### RAII and Exception Safety

```cpp
// RAII-based resource management with exception safety
class ExceptionSafeResource {
private:
    std::unique_ptr<int[]> data_;
    std::FILE* file_;
    std::mutex* mutex_;
    
public:
    ExceptionSafeResource(const std::string& filename) 
        : file_(nullptr), mutex_(nullptr) {
        try {
            // Acquire resources in order that allows proper cleanup
            mutex_ = new std::mutex(); // Can throw
            std::lock_guard<std::mutex> lock(*mutex_);
            
            file_ = std::fopen(filename.c_str(), "r"); // Can throw
            if (!file_) {
                throw std::runtime_error("Failed to open file");
            }
            
            // More resource acquisition...
            data_ = std::make_unique<int[]>(1000); // Can throw
        } catch (...) {
            // Clean up already acquired resources
            if (file_) {
                std::fclose(file_);
            }
            delete mutex_;
            throw; // Re-throw
        }
    }
    
    ~ExceptionSafeResource() noexcept {
        // Destructors should never throw
        if (file_) {
            std::fclose(file_);
        }
        delete mutex_;
    }
    
    // Move constructor for exception safety
    ExceptionSafeResource(ExceptionSafeResource&& other) noexcept
        : data_(std::move(other.data_)), 
          file_(other.file_), 
          mutex_(other.mutex_) {
        other.file_ = nullptr;
        other.mutex_ = nullptr;
    }
    
    // Move assignment with strong exception safety
    ExceptionSafeResource& operator=(ExceptionSafeResource&& other) noexcept {
        if (this != &other) {
            // Clean up current resources
            if (file_) {
                std::fclose(file_);
            }
            delete mutex_;
            
            // Transfer resources
            data_ = std::move(other.data_);
            file_ = other.file_;
            mutex_ = other.mutex_;
            
            // Reset other
            other.file_ = nullptr;
            other.mutex_ = nullptr;
        }
        return *this;
    }
};
```

### Custom Exception-Safe Smart Pointers

```cpp
// Exception-safe custom smart pointer
template<typename T>
class ExceptionSafeUniquePtr {
private:
    T* ptr_;
    std::function<void(T*)> deleter_;
    
public:
    explicit ExceptionSafeUniquePtr(T* ptr = nullptr, 
                                   std::function<void(T*)> deleter = std::default_delete<T>{})
        : ptr_(ptr), deleter_(std::move(deleter)) {}
    
    // Move constructor with strong exception safety
    ExceptionSafeUniquePtr(ExceptionSafeUniquePtr&& other) noexcept
        : ptr_(other.ptr_), deleter_(std::move(other.deleter_)) {
        other.ptr_ = nullptr;
    }
    
    // Move assignment with strong exception safety
    ExceptionSafeUniquePtr& operator=(ExceptionSafeUniquePtr&& other) noexcept {
        if (this != &other) {
            reset(); // Strong exception safety: no throw
            ptr_ = other.ptr_;
            deleter_ = std::move(other.deleter_);
            other.ptr_ = nullptr;
        }
        return *this;
    }
    
    // Exception-safe reset
    void reset(T* ptr = nullptr) noexcept {
        if (ptr_ != ptr) {
            T* old_ptr = ptr_;
            ptr_ = ptr;
            if (old_ptr) {
                deleter_(old_ptr);
            }
        }
    }
    
    ~ExceptionSafeUniquePtr() noexcept {
        if (ptr_) {
            deleter_(ptr_);
        }
    }
    
    T* get() const noexcept { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    
    // Release ownership without deleting
    T* release() noexcept {
        T* temp = ptr_;
        ptr_ = nullptr;
        return temp;
    }
};
```

## Transactional Programming

### Exception-Safe Transactions

```cpp
// Transactional approach to exception safety
template<typename T>
class TransactionalContainer {
private:
    std::vector<T> data_;
    
public:
    // Transactional insert with strong exception safety
    template<typename... Args>
    void emplace_transactional(Args&&... args) {
        // Create temporary object (can throw)
        T temp(std::forward<Args>(args)...);
        
        // Only modify container after successful construction
        // This provides strong exception safety
        data_.push_back(std::move(temp));
    }
    
    // Transactional batch operation
    template<typename Iterator>
    void insert_batch_transactional(Iterator first, Iterator last) {
        // Pre-allocate space (can throw)
        std::vector<T> temp(data_);
        
        try {
            // Add new elements (can throw)
            while (first != last) {
                temp.push_back(*first);
                ++first;
            }
        } catch (...) {
            // temp is automatically destroyed, original data_ unchanged
            throw; // Re-throw
        }
        
        // No exceptions beyond this point
        data_ = std::move(temp);
    }
};
```

### Copy-and-Swap with Exception Safety

```cpp
// Copy-and-swap idiom for strong exception safety
class CopyAndSwapExample {
private:
    std::unique_ptr<int[]> data_;
    std::size_t size_;
    
public:
    CopyAndSwapExample(std::size_t size = 0) 
        : data_(size > 0 ? std::make_unique<int[]>(size) : nullptr), 
          size_(size) {}
    
    // Copy constructor (can throw)
    CopyAndSwapExample(const CopyAndSwapExample& other) 
        : data_(other.size_ > 0 ? std::make_unique<int[]>(other.size_) : nullptr),
          size_(other.size_) {
        // Copy data (can throw)
        for (std::size_t i = 0; i < size_; ++i) {
            data_[i] = other.data_[i];
        }
    }
    
    // Copy assignment using copy-and-swap (strong exception safety)
    CopyAndSwapExample& operator=(const CopyAndSwapExample& other) {
        // Copy constructor might throw, but *this remains unchanged
        CopyAndSwapExample temp(other);
        
        // swap provides no-throw guarantee
        swap(temp);
        
        return *this;
    }
    
    // Exception-safe swap
    void swap(CopyAndSwapExample& other) noexcept {
        data_.swap(other.data_);
        std::swap(size_, other.size_);
    }
    
    // Friend swap function
    friend void swap(CopyAndSwapExample& first, CopyAndSwapExample& second) noexcept {
        first.swap(second);
    }
};
```

## Exception Safety in Templates

### Exception-Safe Template Design

```cpp
// Template with exception safety considerations
template<typename T, typename Allocator = std::allocator<T>>
class ExceptionSafeTemplate {
private:
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    Allocator alloc_;
    
    // Check if operations provide no-throw guarantee
    static constexpr bool nothrow_move_constructible = 
        std::is_nothrow_move_constructible_v<T>;
        
    static constexpr bool nothrow_move_assignable = 
        std::is_nothrow_move_assignable_v<T>;
    
public:
    // Exception-safe constructor
    ExceptionSafeTemplate(std::size_t count, const T& value = T{}) 
        : data_(nullptr), size_(0), capacity_(0) {
        reserve(count);
        
        // Use guard for exception safety
        struct Guard {
            ExceptionSafeTemplate& container;
            std::size_t constructed_count = 0;
            
            ~Guard() {
                if (constructed_count > 0) {
                    for (std::size_t i = 0; i < constructed_count; ++i) {
                        std::allocator_traits<Allocator>::destroy(
                            container.alloc_, container.data_ + i);
                    }
                }
            }
        } guard{*this};
        
        try {
            for (std::size_t i = 0; i < count; ++i) {
                std::allocator_traits<Allocator>::construct(
                    alloc_, data_ + i, value);
                ++guard.constructed_count;
            }
        } catch (...) {
            // Guard destructor handles cleanup
            throw;
        }
        
        size_ = count;
    }
    
    // Exception-safe reserve
    void reserve(std::size_t new_capacity) {
        if (new_capacity <= capacity_) return;
        
        T* new_data = std::allocator_traits<Allocator>::allocate(alloc_, new_capacity);
        
        // Use different strategies based on T's exception safety properties
        struct Guard {
            T* data;
            std::size_t count;
            Allocator& alloc;
            
            ~Guard() {
                if (data) {
                    for (std::size_t i = 0; i < count; ++i) {
                        std::allocator_traits<Allocator>::destroy(alloc, data + i);
                    }
                    std::allocator_traits<Allocator>::deallocate(alloc, data, count);
                }
            }
        } guard{new_data, 0, alloc_};
        
        try {
            if constexpr (nothrow_move_constructible) {
                // If move is no-throw, we can move directly
                for (std::size_t i = 0; i < size_; ++i) {
                    std::allocator_traits<Allocator>::construct(
                        alloc_, new_data + i, std::move(data_[i]));
                    ++guard.count;
                }
            } else {
                // If move can throw, copy to maintain strong exception safety
                for (std::size_t i = 0; i < size_; ++i) {
                    std::allocator_traits<Allocator>::construct(
                        alloc_, new_data + i, data_[i]);
                    ++guard.count;
                }
            }
        } catch (...) {
            // Guard destructor handles cleanup
            throw;
        }
        
        // No exceptions beyond this point
        for (std::size_t i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        }
        std::allocator_traits<Allocator>::deallocate(alloc_, data_, capacity_);
        
        data_ = new_data;
        capacity_ = new_capacity;
        guard.data = nullptr; // Prevent guard from deallocating
    }
};
```

## Exception Safety in STL Algorithms

### Algorithm Exception Safety

```cpp
// Custom algorithm with exception safety considerations
template<typename Iterator, typename Function>
void exception_safe_for_each(Iterator first, Iterator last, Function f) {
    // Basic exception safety: no resources leaked
    while (first != last) {
        try {
            f(*first);
        } catch (...) {
            // Could provide strong exception safety by rolling back,
            // but that would require additional complexity
            // For now, we provide basic exception safety
            throw;
        }
        ++first;
    }
}

// Exception-safe transform with rollback capability
template<typename InputIterator, typename OutputIterator, typename Function>
OutputIterator exception_safe_transform(InputIterator first, InputIterator last,
                                       OutputIterator result, Function f) {
    OutputIterator original_result = result;
    std::vector<typename std::iterator_traits<OutputIterator>::value_type> temp_storage;
    
    try {
        // Store results in temporary container (can throw)
        while (first != last) {
            temp_storage.push_back(f(*first));
            ++first;
        }
    } catch (...) {
        // If exception occurs, no side effects on output range
        throw;
    }
    
    // Only modify output range after successful computation
    // This provides strong exception safety
    for (const auto& value : temp_storage) {
        *result = value;
        ++result;
    }
    
    return result;
}
```

## Exception Safety Testing

### Testing Exception Safety

```cpp
// Exception safety testing framework
class ExceptionSafetyTester {
private:
    std::size_t throw_on_;
    std::size_t operation_count_;
    
public:
    ExceptionSafetyTester(std::size_t throw_on = 0) 
        : throw_on_(throw_on), operation_count_(0) {}
    
    void maybe_throw() {
        ++operation_count_;
        if (operation_count_ == throw_on_) {
            throw std::runtime_error("Test exception");
        }
    }
    
    void reset() {
        operation_count_ = 0;
    }
};

// Test class that can throw during operations
class TestClass {
private:
    static ExceptionSafetyTester* tester_;
    int value_;
    
public:
    TestClass(int value = 0) : value_(value) {
        if (tester_) tester_->maybe_throw();
    }
    
    TestClass(const TestClass& other) : value_(other.value_) {
        if (tester_) tester_->maybe_throw();
    }
    
    TestClass& operator=(const TestClass& other) {
        if (tester_) tester_->maybe_throw();
        value_ = other.value_;
        return *this;
    }
    
    static void set_tester(ExceptionSafetyTester* tester) {
        tester_ = tester;
    }
};

ExceptionSafetyTester* TestClass::tester_ = nullptr;

// Exception safety test for container operations
void test_container_exception_safety() {
    for (std::size_t throw_point = 1; throw_point <= 10; ++throw_point) {
        ExceptionSafetyTester tester(throw_point);
        TestClass::set_tester(&tester);
        
        try {
            // Test operation that should provide strong exception safety
            std::vector<TestClass> vec;
            vec.reserve(5);
            
            // Add elements
            for (int i = 0; i < 5; ++i) {
                vec.push_back(TestClass(i));
            }
            
            // This operation might throw
            vec.push_back(TestClass(42));
            
            // If we reach here, no exception occurred
            tester.reset();
        } catch (const std::exception&) {
            // Exception occurred, but container should be in valid state
            tester.reset();
            // Continue testing other throw points
        }
    }
}
```

## Best Practices for Exception Safety

### Exception Safety Guidelines

1. **Prefer Strong Exception Safety** - When possible, ensure operations are atomic
2. **Use RAII for Resource Management** - Automatic cleanup during stack unwinding
3. **Implement Copy-and-Swap** - Provides strong exception safety for assignment
4. **Consider No-Throw Operations** - Use noexcept when appropriate
5. **Test Exception Paths** - Ensure exception handling code works correctly
6. **Document Exception Safety** - Clearly indicate guarantees provided
7. **Handle Partial Failures** - Clean up partially constructed objects
8. **Use Exception-Safe Standard Library** - Leverage STL's exception safety

### Exception Safety Patterns

```cpp
// Function with strong exception safety
template<typename T>
void strong_exception_safe_function(std::vector<T>& container, const T& value) {
    // Create temporary container (can throw)
    std::vector<T> temp(container);
    
    try {
        // Modify temporary (can throw)
        temp.push_back(value);
        // More operations...
    } catch (...) {
        // temp is automatically destroyed, container unchanged
        throw; // Re-throw
    }
    
    // No exceptions beyond this point
    container = std::move(temp);
}

// Function with basic exception safety
template<typename T>
void basic_exception_safe_function(std::vector<T>& container, const T& value) {
    // Directly modify container (basic exception safety)
    container.push_back(value);
    // If this throws, container state may be modified but no resources leaked
}
```

Understanding advanced exception safety techniques is crucial for developing robust C++ applications and libraries that maintain their invariants even in the presence of exceptions.