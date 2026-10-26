# Exception Handling Mechanisms

## Introduction

Exception handling is a fundamental mechanism in C++ for dealing with error conditions and exceptional circumstances in a structured and controlled manner. Understanding how exceptions work internally is crucial for writing robust, efficient code and for comprehending the implementation of exception safety in the STL.

## Exception Handling Fundamentals

### Basic Exception Syntax

```cpp
// Throwing exceptions
void function_that_can_fail() {
    if (/* some error condition */) {
        throw std::runtime_error("Something went wrong!");
    }
}

// Catching exceptions
try {
    function_that_can_fail();
} catch (const std::runtime_error& e) {
    // Handle specific exception type
    std::cerr << "Runtime error: " << e.what() << std::endl;
} catch (const std::exception& e) {
    // Handle any standard exception
    std::cerr << "Standard exception: " << e.what() << std::endl;
} catch (...) {
    // Handle any other exception
    std::cerr << "Unknown exception occurred" << std::endl;
}
```

### Exception Objects

```cpp
// Custom exception class
class CustomException : public std::exception {
private:
    std::string message_;
    
public:
    explicit CustomException(const std::string& msg) : message_(msg) {}
    
    const char* what() const noexcept override {
        return message_.c_str();
    }
};

// Throwing with different value categories
void demonstrate_exception_values() {
    // Throwing temporary (rvalue)
    throw std::runtime_error("temporary exception");
    
    // Throwing local variable (lvalue) - gets copied
    std::runtime_error local_error("local exception");
    throw local_error; // Copy constructor called
    
    // Throwing reference to object - gets copied
    std::runtime_error& ref_error = local_error;
    throw ref_error; // Copy constructor called
}
```

## Stack Unwinding

### The Unwinding Process

When an exception is thrown, the C++ runtime performs stack unwinding:

```cpp
struct UnwindingTracker {
    int id_;
    
    UnwindingTracker(int id) : id_(id) {
        std::cout << "Constructing " << id_ << std::endl;
    }
    
    ~UnwindingTracker() {
        std::cout << "Destroying " << id_ << std::endl;
    }
};

void level3() {
    UnwindingTracker t3(3);
    throw std::runtime_error("Exception in level3");
    // t3 destructor called during stack unwinding
}

void level2() {
    UnwindingTracker t2(2);
    level3();
    // This code never executes
}

void level1() {
    UnwindingTracker t1(1);
    try {
        level2();
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << std::endl;
    }
    // t1 destructor called normally
}

// Output:
// Constructing 1
// Constructing 2
// Constructing 3
// Destroying 3
// Destroying 2
// Caught: Exception in level3
// Destroying 1
```

### Exception Safety Guarantees

```cpp
class ExceptionSafeContainer {
private:
    std::unique_ptr<int[]> data_;
    std::size_t size_;
    std::size_t capacity_;
    
public:
    // Strong exception safety guarantee
    void push_back(int value) {
        if (size_ == capacity_) {
            // Create new container with increased capacity
            // This operation can throw, but original object remains unchanged
            std::size_t new_capacity = capacity_ == 0 ? 1 : capacity_ * 2;
            auto new_data = std::make_unique<int[]>(new_capacity);
            
            // Copy existing elements (can throw, but original unchanged)
            for (std::size_t i = 0; i < size_; ++i) {
                new_data[i] = data_[i];
            }
            
            // Only modify object state after all operations that can throw
            data_ = std::move(new_data);
            capacity_ = new_capacity;
        }
        
        // This operation can throw, but we've already allocated space
        data_[size_] = value; // Assignment might throw
        ++size_;
    }
    
    // No-throw guarantee
    void swap(ExceptionSafeContainer& other) noexcept {
        data_.swap(other.data_);
        std::swap(size_, other.size_);
        std::swap(capacity_, other.capacity_);
    }
};
```

## Exception Handling Implementation

### Compiler-Generated Code

The compiler transforms exception handling code into a state machine:

```cpp
// Original code with exception handling
int original_function() {
    int x = 10;
    try {
        might_throw();
    } catch (const std::exception& e) {
        x = 20;
        return x;
    }
    x = 30;
    return x;
}

// Conceptual transformation by compiler
int transformed_function() {
    int x = 10;
    int exception_state = 0;
    
    // Set up exception handling frame
    try {
        might_throw();
        exception_state = 1; // Success path
    } catch (const std::exception& e) {
        // Exception path
        x = 20;
        return x;
    }
    
    // Continue with normal execution
    x = 30;
    return x;
}
```

### Exception Tables

The compiler generates exception tables that map instruction pointers to handler information:

```cpp
// Simplified representation of exception table entries
struct ExceptionTableEntry {
    void* start_address;     // Start of try block
    void* end_address;       // End of try block
    void* handler_address;   // Address of catch handler
    std::type_info* type_info; // Type information for caught exception
    int stack_adjustment;    // Stack adjustment needed
};

// Runtime uses these tables to determine which handler to call
// when an exception is thrown
```

## Exception Handling Performance

### Cost of Exception Handling

Exception handling has both compile-time and runtime costs:

```cpp
// No exception handling - fastest execution
void fast_function_noexcept() noexcept {
    // Direct execution path
    perform_operation();
}

// With exception handling - has overhead even when no exception occurs
void function_with_exceptions() {
    try {
        perform_operation();
    } catch (const std::exception& e) {
        handle_error(e);
    }
}

// Exception specification optimization
void optimized_function() noexcept {
    // Compiler can optimize more aggressively
    // because it knows no exceptions will be thrown
    perform_optimized_operation();
}
```

### Exception Handling Overhead

```cpp
// Measuring exception handling overhead
#include <chrono>

void benchmark_exception_overhead() {
    const int iterations = 1000000;
    
    // Test without exceptions
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        fast_function_noexcept();
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto no_exception_time = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    // Test with exceptions (but no exceptions thrown)
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        function_with_exceptions();
    }
    end = std::chrono::high_resolution_clock::now();
    auto with_exception_time = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::cout << "No exception time: " << no_exception_time.count() << " microseconds\n";
    std::cout << "With exception time: " << with_exception_time.count() << " microseconds\n";
    std::cout << "Overhead: " << (with_exception_time.count() - no_exception_time.count()) * 100.0 / no_exception_time.count() << "%\n";
}
```

## Advanced Exception Handling

### Function Try Blocks

```cpp
// Function try blocks for constructors and destructors
class ComplexClass {
private:
    std::unique_ptr<int[]> data_;
    int size_;
    
public:
    // Function try block for constructor
    ComplexClass(int size) try 
        : data_(std::make_unique<int[]>(size)), size_(size) {
        // Constructor body
        initialize_data();
    } catch (...) {
        // Handle exceptions from member initialization or constructor body
        std::cerr << "Constructor failed, cleaning up..." << std::endl;
        // Note: Members are automatically destroyed before this handler runs
        throw; // Re-throw the exception
    }
    
    // Function try block for destructor
    ~ComplexClass() noexcept(false) try {
        // Destructor body
        cleanup_resources();
    } catch (...) {
        // Handle exceptions from destructor
        std::cerr << "Destructor failed!" << std::endl;
        // Note: Throwing from destructor is generally discouraged
        // This is just for demonstration
    }
};
```

### Exception Specifications

```cpp
// Dynamic exception specification (deprecated in C++11, removed in C++17)
void old_style_function() throw(std::runtime_error, std::logic_error) {
    // Can only throw std::runtime_error or std::logic_error
    // Any other exception would call std::unexpected
}

// Noexcept specification (C++11)
void modern_function() noexcept {
    // Guaranteed not to throw
    // Compiler can optimize more aggressively
}

void conditional_noexcept_function(int value) noexcept(noexcept(std::declval<int>() + std::declval<int>())) {
    // noexcept if the operation doesn't throw
    int result = value + 10;
}

// Noexcept with templates
template<typename T>
void template_function(T&& value) noexcept(noexcept(T(std::forward<T>(value)))) {
    T copy(std::forward<T>(value));
}
```

### Exception Safety in Templates

```cpp
// Exception-safe template implementation
template<typename T, typename Allocator = std::allocator<T>>
class exception_safe_vector {
private:
    T* data_;
    std::size_t size_;
    std::size_t capacity_;
    Allocator alloc_;
    
    void reallocate(std::size_t new_capacity) {
        // Strong exception safety guarantee
        auto new_data = std::allocator_traits<Allocator>::allocate(alloc_, new_capacity);
        
        // Use guard to ensure cleanup if exception occurs
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
            // Copy construct elements (can throw)
            for (std::size_t i = 0; i < size_; ++i) {
                std::allocator_traits<Allocator>::construct(alloc_, new_data + i, data_[i]);
                ++guard.count;
            }
        } catch (...) {
            // Guard destructor automatically cleans up new_data
            throw; // Re-throw the exception
        }
        
        // No exceptions beyond this point
        // Clean up old data
        for (std::size_t i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        }
        std::allocator_traits<Allocator>::deallocate(alloc_, data_, capacity_);
        
        // Update object state
        data_ = new_data;
        capacity_ = new_capacity;
        guard.data = nullptr; // Prevent guard from deallocating
    }
    
public:
    void push_back(const T& value) {
        if (size_ == capacity_) {
            reallocate(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        
        std::allocator_traits<Allocator>::construct(alloc_, data_ + size_, value);
        ++size_;
    }
    
    void push_back(T&& value) {
        if (size_ == capacity_) {
            reallocate(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        
        std::allocator_traits<Allocator>::construct(alloc_, data_ + size_, std::move(value));
        ++size_;
    }
};
```

## Exception Handling in STL

### Exception Safety in Standard Containers

```cpp
// STL containers provide different exception safety guarantees:
// 1. No-throw guarantee - Operations never throw
// 2. Strong guarantee - Operations succeed completely or have no effect
// 3. Basic guarantee - No resources are leaked, but object state may be modified

class stl_exception_safety_examples {
public:
    void vector_examples() {
        std::vector<int> vec;
        
        // push_back provides strong exception safety guarantee
        // If reallocation is needed and fails, vector remains unchanged
        vec.push_back(42);
        
        // swap provides no-throw guarantee
        std::vector<int> other_vec;
        vec.swap(other_vec); // No exceptions thrown
        
        // clear provides no-throw guarantee
        vec.clear(); // No exceptions thrown
    }
    
    void map_examples() {
        std::map<int, std::string> map;
        
        // insert provides strong exception safety guarantee
        // If insertion fails, map remains unchanged
        map.insert({1, "one"});
        
        // operator[] provides strong exception safety for construction
        map[2] = "two"; // May throw, but map state remains valid
    }
};
```

### Exception Propagation in STL Algorithms

```cpp
// STL algorithms may propagate exceptions from user-provided functions
void algorithm_exception_examples() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    try {
        // std::transform may propagate exceptions from the function
        std::transform(vec.begin(), vec.end(), vec.begin(), 
                      [](int x) {
                          if (x == 3) {
                              throw std::runtime_error("Don't like 3!");
                          }
                          return x * 2;
                      });
    } catch (const std::exception& e) {
        // Exception from lambda is propagated through transform
        std::cout << "Caught exception: " << e.what() << std::endl;
        // vec may be partially modified
    }
}
```

## Best Practices

### Exception Handling Guidelines

1. **Use Exceptions for Exceptional Conditions** - Don't use exceptions for normal control flow
2. **Throw by Value, Catch by Reference** - Prevent object slicing and unnecessary copies
3. **Provide Strong Exception Safety** - When possible, ensure operations are atomic
4. **Use RAII for Resource Management** - Automatic cleanup during stack unwinding
5. **Don't Throw from Destructors** - Destructors should be noexcept
6. **Document Exception Specifications** - Clearly indicate which functions can throw
7. **Minimize Exception Handling Overhead** - Use noexcept when exceptions won't occur
8. **Test Exception Paths** - Ensure exception handling code works correctly

### Exception Safety Patterns

```cpp
// Copy-and-swap idiom for strong exception safety
class SafeClass {
private:
    // Implementation details
    
public:
    SafeClass& operator=(const SafeClass& other) {
        // Copy constructor might throw, but *this remains unchanged
        SafeClass temp(other);
        
        // swap provides no-throw guarantee
        swap(temp);
        
        return *this;
    }
    
    void swap(SafeClass& other) noexcept {
        // Implementation that doesn't throw
    }
};

// Function try blocks for constructors
class ResourceClass {
public:
    ResourceClass() try {
        // Acquire resources
        acquire_resources();
    } catch (...) {
        // Clean up partially acquired resources
        release_resources();
        throw; // Re-throw
    }
};
```

Understanding exception handling mechanisms is essential for writing robust C++ code and for comprehending how the STL provides exception safety guarantees.