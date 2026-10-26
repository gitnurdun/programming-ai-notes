# Modern C++ Features

## Introduction

Modern C++ features introduced in C++11 and beyond have transformed the language, making it more expressive, efficient, and safer. These features enable developers to write cleaner, more maintainable code while leveraging the full power of the language. Understanding these features is crucial for mastering contemporary C++ development.

## Core Language Enhancements

### Auto and Type Inference

```cpp
// Modern type inference with auto
class TypeInferenceExamples {
public:
    // Basic auto usage
    void basic_auto_examples() {
        auto x = 42;                    // x is int
        auto y = 3.14;                  // y is double
        auto z = "hello";               // z is const char*
        
        const auto& ref = x;            // ref is const int&
        auto&& universal_ref = x;       // universal_ref is int& (collapsed)
        auto&& universal_ref2 = 42;     // universal_ref2 is int&&
    }
    
    // Trailing return type
    template<typename T, typename U>
    auto multiply(T&& t, U&& u) -> decltype(std::forward<T>(t) * std::forward<U>(u)) {
        return std::forward<T>(t) * std::forward<U>(u);
    }
    
    // C++14 generic lambdas with auto
    void generic_lambdas() {
        auto process = [](auto&&... args) {
            return (args + ... + 0);  // C++17 fold expression
        };
        
        auto result1 = process(1, 2.0, 3);     // Returns 6.0
        auto result2 = process(1.5f, 2.5f);    // Returns 4.0f
    }
    
    // Decltype for complex expressions
    template<typename Container>
    auto get_element(Container&& c, std::size_t index) -> 
        decltype(std::forward<Container>(c)[index]) {
        return std::forward<Container>(c)[index];
    }
};
```

### Range-Based For Loops

```cpp
// Advanced range-based for loop usage
class RangeBasedForExamples {
public:
    void advanced_range_for() {
        std::vector<int> vec = {1, 2, 3, 4, 5};
        
        // Basic usage
        for (const auto& element : vec) {
            std::cout << element << " ";
        }
        
        // With initializer (C++20)
        for (const auto& [key, value] : std::map<int, std::string>{{1, "one"}, {2, "two"}}) {
            std::cout << key << ": " << value << std::endl;
        }
        
        // Custom range with structured bindings
        std::vector<std::pair<int, std::string>> pairs = {{1, "first"}, {2, "second"}};
        
        for (const auto& [num, str] : pairs) {
            std::cout << num << " -> " << str << std::endl;
        }
    }
    
    // Working with custom ranges
    template<typename Range>
    void process_range(Range&& range) {
        for (auto&& element : range) {
            // Universal reference allows processing of any range type
            std::cout << element << " ";
        }
    }
};
```

## Smart Pointers and RAII

### Advanced Smart Pointer Usage

```cpp
#include <memory>
#include <shared_mutex>

// Advanced smart pointer patterns
class SmartPointerPatterns {
public:
    // Custom deleter example
    void custom_deleter_example() {
        // File handle with custom deleter
        auto file_deleter = [](FILE* f) {
            if (f) {
                std::fclose(f);
            }
        };
        
        std::unique_ptr<FILE, decltype(file_deleter)> file(
            std::fopen("example.txt", "r"), file_deleter);
        
        if (file) {
            // Use file...
        }
        // File automatically closed when unique_ptr goes out of scope
    }
    
    // Weak pointer for breaking circular references
    class Node {
    public:
        std::shared_ptr<Node> next;
        std::weak_ptr<Node> parent;  // Breaks circular reference
        int value;
        
        Node(int val) : value(val) {}
        
        ~Node() {
            std::cout << "Node " << value << " destroyed" << std::endl;
        }
    };
    
    // Shared pointer with custom allocator
    template<typename T, typename Allocator = std::allocator<T>>
    std::shared_ptr<T> allocate_shared_with_allocator(Allocator alloc = Allocator{}) {
        return std::allocate_shared<T>(alloc, T{});
    }
    
    // Atomic shared pointer (C++20)
    void atomic_shared_ptr_example() {
        std::atomic<std::shared_ptr<int>> atomic_sp = std::make_shared<int>(42);
        
        // Thread-safe operations
        auto current = atomic_sp.load();
        atomic_sp.store(std::make_shared<int>(100));
        
        // Compare and swap
        auto expected = current;
        auto desired = std::make_shared<int>(200);
        atomic_sp.compare_exchange_weak(expected, desired);
    }
    
    // Observer pattern with weak pointer
    template<typename T>
    class Observable {
    private:
        std::vector<std::weak_ptr<T>> observers_;
        std::mutex mutex_;
        
    public:
        void add_observer(std::shared_ptr<T> observer) {
            std::lock_guard<std::mutex> lock(mutex_);
            observers_.push_back(observer);
        }
        
        template<typename Func>
        void notify_observers(Func&& func) {
            std::lock_guard<std::mutex> lock(mutex_);
            
            // Remove expired weak pointers and notify valid ones
            observers_.erase(
                std::remove_if(observers_.begin(), observers_.end(),
                    [&func](const std::weak_ptr<T>& wp) {
                        if (auto sp = wp.lock()) {
                            func(sp);
                            return false;  // Keep valid observers
                        }
                        return true;  // Remove expired observers
                    }),
                observers_.end()
            );
        }
    };
};
```

## Lambda Functions and Functional Programming

### Advanced Lambda Techniques

```cpp
// Advanced lambda patterns and functional programming
class LambdaAdvancedPatterns {
public:
    // Recursive lambda with auto
    void recursive_lambda() {
        // Method 1: Using std::function
        std::function<int(int)> factorial = [&factorial](int n) -> int {
            return (n <= 1) ? 1 : n * factorial(n - 1);
        };
        
        // Method 2: Generic lambda with auto
        auto fibonacci = [](auto&& fib_ref) -> auto {
            return [fib_ref](int n) -> int {
                return (n <= 1) ? n : fib_ref(fib_ref)(n-1) + fib_ref(fib_ref)(n-2);
            };
        };
        
        auto fib = fibonacci(fibonacci);
        std::cout << "fib(10) = " << fib(10) << std::endl;
    }
    
    // Lambda with perfect forwarding
    template<typename Func>
    auto make_forwarder(Func&& func) {
        return [func = std::forward<Func>(func)](auto&&... args) {
            return func(std::forward<decltype(args)>(args)...);
        };
    }
    
    // Lambda with capture by value vs reference
    void capture_examples() {
        int x = 42;
        int y = 100;
        
        // Capture by value
        auto by_value = [x]() { return x; };
        
        // Capture by reference
        auto by_ref = [&y]() { return y; };
        
        // Capture all by value
        auto all_by_value = [=]() { return x + y; };
        
        // Capture all by reference
        auto all_by_ref = [&]() { return x + y; };
        
        // Capture with move (C++14)
        std::unique_ptr<int> ptr = std::make_unique<int>(42);
        auto moved_capture = [moved_ptr = std::move(ptr)]() {
            return *moved_ptr;
        };
    }
    
    // Higher-order functions with lambdas
    template<typename Container, typename Predicate>
    auto filter(const Container& container, Predicate pred) {
        std::vector<typename Container::value_type> result;
        std::copy_if(container.begin(), container.end(), 
                    std::back_inserter(result), pred);
        return result;
    }
    
    template<typename Container, typename Func>
    auto map(const Container& container, Func func) {
        using result_type = decltype(func(std::declval<typename Container::value_type>()));
        std::vector<result_type> result;
        result.reserve(container.size());
        
        std::transform(container.begin(), container.end(), 
                      std::back_inserter(result), func);
        return result;
    }
    
    // Usage example
    void functional_example() {
        std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        
        auto evens = filter(numbers, [](int n) { return n % 2 == 0; });
        auto squares = map(evens, [](int n) { return n * n; });
        
        for (int square : squares) {
            std::cout << square << " ";
        }
        std::cout << std::endl;
    }
};
```

## Ranges (C++20)

### Ranges and Views

```cpp
#include <ranges>
#include <vector>
#include <algorithm>
#include <numeric>

// Modern ranges usage
class RangesExamples {
public:
    void basic_ranges_usage() {
        std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        
        // Filter and transform with ranges
        auto result = numbers 
                    | std::views::filter([](int n) { return n % 2 == 0; })
                    | std::views::transform([](int n) { return n * n; })
                    | std::views::take(3);
        
        for (int value : result) {
            std::cout << value << " ";
        }
        std::cout << std::endl;
    }
    
    // Custom range adapter
    template<std::ranges::range R>
    auto square_view(R&& range) {
        return std::views::transform(std::forward<R>(range), 
                                   [](const auto& x) { return x * x; });
    }
    
    // Range-based algorithms
    void range_algorithms() {
        std::vector<int> vec = {1, 2, 3, 4, 5};
        
        // Using ranges algorithms
        auto sum = std::ranges::fold_left(vec, 0, std::plus{});
        
        // Find if any element satisfies condition
        bool has_even = std::ranges::any_of(vec, [](int x) { return x % 2 == 0; });
        
        // Sort with ranges
        std::ranges::sort(vec, std::greater{});
        
        // Partition with ranges
        auto [even_it, odd_it] = std::ranges::partition_point(
            vec, [](int x) { return x % 2 == 0; });
    }
    
    // Infinite ranges (C++23 concept)
    void infinite_range_like() {
        // Create a range of squares
        auto squares = std::views::iota(0) 
                     | std::views::transform([](int i) { return i * i; })
                     | std::views::take(10);
        
        for (int square : squares) {
            std::cout << square << " ";
        }
        std::cout << std::endl;
    }
    
    // Range composition
    template<std::ranges::range R>
    auto process_range(R&& r) {
        return std::forward<R>(r)
             | std::views::filter([](auto&& x) { return x > 0; })
             | std::views::transform([](auto&& x) { return x * 2; })
             | std::views::take(5);
    }
};
```

## Concepts and Template Metaprogramming

### Modern Template Techniques

```cpp
#include <concepts>
#include <type_traits>

// Modern template metaprogramming with concepts
class ModernTemplatePatterns {
public:
    // Basic concepts
    template<typename T>
    concept Integral = std::is_integral_v<T>;
    
    template<typename T>
    concept Arithmetic = std::is_arithmetic_v<T>;
    
    template<typename T>
    concept Iterable = requires(T t) {
        t.begin();
        t.end();
        { t.begin() != t.end() } -> std::convertible_to<bool>;
    };
    
    // Constrained templates
    template<Arithmetic T>
    T calculate(T a, T b) {
        return a + b;
    }
    
    // Advanced concepts with requirements
    template<typename T>
    concept Callable = requires(T t) {
        t();
        requires std::is_same_v<decltype(t()), int>;
    };
    
    template<typename T>
    concept ResizableContainer = requires(T t) {
        t.resize(std::size_t{});
        { t.size() } -> std::convertible_to<std::size_t>;
    };
    
    // Abbreviated function templates (C++20)
    void process_integers(Integral auto value) {
        std::cout << "Processing integer: " << value << std::endl;
    }
    
    void process_arithmetic(Arithmetic auto value) {
        std::cout << "Processing arithmetic: " << value << std::endl;
    }
    
    // Template template parameters with constraints
    template<template<typename> typename Container, Arithmetic T>
    class ConstrainedContainer {
    private:
        Container<T> data_;
        
    public:
        void add(T value) {
            data_.push_back(value);
        }
        
        T sum() const {
            T result = T{};
            for (const auto& item : data_) {
                result += item;
            }
            return result;
        }
    };
    
    // Conditional compilation with concepts
    template<typename T>
    void conditional_function(T&& t) {
        if constexpr (std::is_arithmetic_v<std::decay_t<T>>) {
            std::cout << "Arithmetic type: " << t << std::endl;
        } else if constexpr (std::is_same_v<std::decay_t<T>, std::string>) {
            std::cout << "String type: " << t << std::endl;
        } else {
            std::cout << "Other type" << std::endl;
        }
    }
    
    // Requires expressions for complex constraints
    template<typename T>
    concept IteratorPair = requires(T t) {
        typename std::iterator_traits<T>::value_type;
        { *t } -> std::convertible_to<typename std::iterator_traits<T>::value_type>;
        ++t;
        { t++ } -> std::convertible_to<T>;
    };
};
```

## Coroutines (C++20)

### Coroutine Implementation

```cpp
#include <coroutine>
#include <exception>

// Basic coroutine promise and return types
template<typename T>
class Generator {
public:
    struct promise_type {
        T value_;
        Generator<T> get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        
        std::suspend_always yield_value(T value) {
            value_ = value;
            return {};
        }
        
        void return_void() {}
        
        void unhandled_exception() {
            std::terminate();
        }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}
    
    ~Generator() {
        if (handle_) handle_.destroy();
    }
    
    Generator(const Generator&) = delete;
    Generator& operator=(const Generator&) = delete;
    
    Generator(Generator&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
    
    Generator& operator=(Generator&& other) noexcept {
        if (this != &other) {
            if (handle_) handle_.destroy();
            handle_ = other.handle_;
            other.handle_ = nullptr;
        }
        return *this;
    }
    
    T operator()() {
        handle_.resume();
        return handle_.promise().value_;
    }
    
    explicit operator bool() const {
        return handle_ && !handle_.done();
    }
    
    T value() const {
        return handle_.promise().value_;
    }
    
    void next() {
        if (handle_) handle_.resume();
    }
};

// Coroutine usage example
Generator<int> fibonacci_generator() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        int temp = a + b;
        a = b;
        b = temp;
    }
}

// Async coroutine example
template<typename T>
class Task {
public:
    struct promise_type {
        T value_;
        
        Task<T> get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        
        void return_value(T value) {
            value_ = value;
        }
        
        void unhandled_exception() {
            std::terminate();
        }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    explicit Task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    
    ~Task() {
        if (handle_) handle_.destroy();
    }
    
    T get() {
        handle_.resume();
        return handle_.promise().value_;
    }
};

Task<int> async_computation() {
    co_return 42;
}
```

## Modules (C++20)

### Modern Module Usage

```cpp
// Example of how modules would be structured
// Note: This is conceptual as modules have complex build requirements

/*
// math_utils.ixx
module;

#include <vector>
#include <numeric>

export module math_utils;

export namespace math {
    template<typename T>
    T square(T x) {
        return x * x;
    }
    
    template<typename Container>
    auto sum(const Container& c) {
        return std::accumulate(c.begin(), c.end(), typename Container::value_type{});
    }
}

export template<typename T>
class Calculator {
public:
    T add(T a, T b) { return a + b; }
    T multiply(T a, T b) { return a * b; }
};
*/

// For now, demonstrating the concept with traditional headers
class ModuleExample {
public:
    // Conceptual module interface
    template<typename T>
    static T square(T x) {
        return x * x;
    }
    
    // Module would provide clean interfaces like this
    template<typename Container>
    static auto calculate_sum(const Container& container) {
        return std::accumulate(container.begin(), container.end(), 
                             typename Container::value_type{});
    }
};
```

## Best Practices

### Modern C++ Guidelines

1. **Prefer RAII** - Use smart pointers and RAII for resource management
2. **Use auto for Type Deduction** - Let the compiler deduce complex types
3. **Embrace Concepts** - Use concepts for better error messages and constraints
4. **Leverage Ranges** - Use ranges for functional-style programming
5. **Use Coroutines for Asynchronous Operations** - For complex async patterns
6. **Prefer Range-Based For Loops** - More readable and safer
7. **Use Lambda Expressions** - For functional programming patterns
8. **Avoid Raw Pointers** - Use smart pointers or references instead

### Migration from Traditional C++

```cpp
// Traditional vs Modern C++ patterns

// Traditional: Manual memory management
class TraditionalExample {
    int* data_;
    std::size_t size_;
    
public:
    TraditionalExample(std::size_t size) : size_(size) {
        data_ = new int[size_];
    }
    
    ~TraditionalExample() {
        delete[] data_;
    }
    
    TraditionalExample(const TraditionalExample& other) 
        : size_(other.size_) {
        data_ = new int[size_];
        std::copy(other.data_, other.data_ + size_, data_);
    }
    
    TraditionalExample& operator=(const TraditionalExample& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new int[size_];
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }
};

// Modern: RAII and smart pointers
class ModernExample {
    std::vector<int> data_;
    
public:
    ModernExample(std::size_t size) : data_(size) {}
    
    // Compiler-generated copy/move operations are correct
    ModernExample(const ModernExample&) = default;
    ModernExample& operator=(const ModernExample&) = default;
    ModernExample(ModernExample&&) = default;
    ModernExample& operator=(ModernExample&&) = default;
    
    // No explicit destructor needed
};
```

Modern C++ features provide powerful tools for writing cleaner, safer, and more expressive code. Understanding and properly applying these features is essential for contemporary C++ development.