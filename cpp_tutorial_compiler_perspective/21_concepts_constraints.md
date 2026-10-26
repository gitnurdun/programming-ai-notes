# Concepts and Constraints

## Introduction

Concepts and constraints are powerful C++20 features that allow specifying requirements for template parameters in a clear, declarative way. They replace complex SFINAE techniques with more readable and maintainable syntax, providing better error messages and enabling more sophisticated generic programming patterns.

## Basic Concepts

### Defining Simple Concepts

```cpp
#include <concepts>
#include <type_traits>

// Basic concept definition
template<typename T>
concept Integral = std::is_integral_v<T>;

// Usage of the concept
template<Integral T>
void process_integral(T value) {
    // Function body - T is guaranteed to be integral
    std::cout << "Processing integral value: " << value << std::endl;
}

// More complex concept with multiple requirements
template<typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template<Arithmetic T>
T add(T a, T b) {
    return a + b;
}

// Using the concepts
process_integral(42);     // OK
// process_integral(3.14); // Compilation error - double is not integral
```

### Concept Definition Syntax

```cpp
// Concept with simple boolean expression
template<typename T>
concept DefaultConstructible = requires {
    T{};  // T{} must be a valid expression
};

// Concept with multiple requirements
template<typename T>
concept Printable = requires(T t) {
    std::cout << t;       // Expression must be valid
    requires std::is_same_v<decltype(std::cout << t), std::ostream&>;
};

// Concept with named requirements
template<typename T>
concept Iterator = requires(T t) {
    typename std::iterator_traits<T>::value_type;  // Nested type requirement
    { *t } -> std::convertible_to<typename std::iterator_traits<T>::value_type>;  // Return type requirement
    ++t;  // Expression requirement
};
```

## Requirements and Constraints

### Simple Requirements

```cpp
// Simple requirement - expression must be valid
template<typename T>
concept HasBegin = requires(T t) {
    t.begin();  // t.begin() must be a valid expression
};

// Usage example
template<HasBegin Container>
void print_first(Container& c) {
    auto it = c.begin();
    std::cout << *it << std::endl;
}

// std::vector satisfies HasBegin, but int does not
std::vector<int> vec = {1, 2, 3};
// print_first(vec);    // OK
// print_first(42);     // Compilation error
```

### Type Requirements

```cpp
// Type requirement - nested type must exist
template<typename T>
concept HasValueType = requires {
    typename T::value_type;
};

// More specific type requirement
template<typename T>
concept HasIterator = requires {
    typename T::iterator;
    requires std::is_same_v<typename T::iterator, typename T::const_iterator>;
};

// Example usage
template<HasValueType Container>
void demonstrate_value_type() {
    typename Container::value_type value{};
    // Use value_type in implementation
}
```

### Compound Requirements

```cpp
// Compound requirement with return type specification
template<typename T>
concept Callable = requires(T t) {
    { t() } -> std::convertible_to<int>;  // t() must be callable and return something convertible to int
};

// Compound requirement with noexcept specification
template<typename T>
concept NoThrowSwappable = requires(T& a, T& b) {
    { std::swap(a, b) } noexcept;  // swap must not throw
};

// Example implementation
template<Callable F>
int call_and_return(F func) {
    return func();
}
```

### Nested Requirements

```cpp
// Nested requirements with constraints
template<typename T>
concept ContainerWithEquality = requires(T t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() } -> std::sentinel_for<decltype(t.begin())>;
    requires requires(typename T::value_type a, typename T::value_type b) {
        { a == b } -> std::convertible_to<bool>;
    };
};
```

## Advanced Concept Patterns

### Concept Refinement

```cpp
// Concept refinement - building more specific concepts from general ones
template<typename T>
concept Copyable = requires(T t) {
    T{t};  // Copy constructible
    t = t; // Copy assignable
};

template<typename T>
concept Movable = requires(T t) {
    T{std::move(t)};  // Move constructible
    t = std::move(t); // Move assignable
};

template<typename T>
concept Regular = Copyable<T> && Movable<T> && std::equality_comparable<T>;

// Usage
template<Regular T>
void process_regular(T value) {
    T copy = value;      // Copyable
    T moved = std::move(value);  // Movable
    if (copy == moved) { // Equality comparable
        // ...
    }
}
```

### Function Template Concepts

```cpp
// Concepts for function objects
template<typename F, typename... Args>
concept CallableWith = requires(F&& f, Args&&... args) {
    f(args...);
};

template<typename F, typename T>
concept Predicate = requires(F&& f, T&& t) {
    { f(t) } -> std::convertible_to<bool>;
};

// Usage in template functions
template<CallableWith<int, double> F>
auto call_with_args(F&& func) {
    return func(42, 3.14);
}

template<Predicate<int> P>
void filter_integers(std::vector<int>& vec, P pred) {
    vec.erase(std::remove_if(vec.begin(), vec.end(), 
               [&pred](int x) { return !pred(x); }), 
              vec.end());
}
```

### Class Template Concepts

```cpp
// Concepts for class templates
template<typename T>
concept NumericType = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
    { a - b } -> std::convertible_to<T>;
    { a * b } -> std::convertible_to<T>;
    { a / b } -> std::convertible_to<T>;
    { a == b } -> std::convertible_to<bool>;
    { a < b } -> std::convertible_to<bool>;
};

template<NumericType T>
class Calculator {
private:
    T value_;
    
public:
    explicit Calculator(T initial = T{}) : value_(initial) {}
    
    Calculator& add(T x) { value_ += x; return *this; }
    Calculator& subtract(T x) { value_ -= x; return *this; }
    Calculator& multiply(T x) { value_ *= x; return *this; }
    Calculator& divide(T x) { value_ /= x; return *this; }
    
    T get_value() const { return value_; }
};
```

## Constraint Syntax

### Requires Clauses

```cpp
// Requires clause on template declaration
template<typename T>
requires Integral<T>
void func1(T value);

// Multiple requirements
template<typename T>
requires Integral<T> && (sizeof(T) >= 4)
void func2(T value);

// Logical combinations
template<typename T>
requires (Integral<T> || FloatingPoint<T>) && DefaultConstructible<T>
void func3(T value);

// Requires clause on function definition
template<typename T>
void func4(T value) requires std::is_arithmetic_v<T> {
    // Function body
}
```

### Requires Expressions

```cpp
// Requires expression as a condition
template<typename T>
constexpr bool is_printable = requires(T t) {
    std::cout << t;
};

// Requires expression in immediate context
template<typename T>
void conditional_func(T value) {
    if constexpr (requires { std::cout << value; }) {
        std::cout << "Printable: " << value << std::endl;
    } else {
        std::cout << "Not printable" << std::endl;
    }
}

// Requires expression in function template
template<typename T>
void requires_expr_func(T value) requires requires { value.some_method(); } {
    value.some_method();
}
```

## Standard Library Concepts

### Iterator Concepts

```cpp
#include <iterator>

// Standard library iterator concepts
template<std::input_iterator Iter>
void process_input_range(Iter first, Iter last) {
    while (first != last) {
        std::cout << *first << " ";
        ++first;
    }
    std::cout << std::endl;
}

template<std::forward_iterator Iter>
void process_forward_range(Iter first, Iter last) {
    // Can use multiple passes
    for (auto it = first; it != last; ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
}

template<std::random_access_iterator Iter>
void process_random_access(Iter first, Iter last) {
    // Can use arithmetic operations
    std::cout << "Size: " << last - first << std::endl;
    for (auto i = 0; i < last - first; ++i) {
        std::cout << first[i] << " ";
    }
    std::cout << std::endl;
}
```

### Callable Concepts

```cpp
#include <functional>

// Standard library callable concepts
template<std::invocable<int, double> F>
auto call_with_types(F&& func) {
    return func(42, 3.14);
}

template<std::predicate<int> P>
void filter_with_predicate(std::vector<int>& vec, P&& pred) {
    vec.erase(std::remove_if(vec.begin(), vec.end(), pred), vec.end());
}

// Comparison concepts
template<std::equality_comparable T>
bool are_equal(const T& a, const T& b) {
    return a == b;
}

template<std::totally_ordered T>
T get_maximum(const T& a, const T& b) {
    return (a < b) ? b : a;
}
```

### Range Concepts

```cpp
#include <ranges>

// Range concepts usage
template<std::ranges::range Range>
void process_range(Range&& r) {
    for (auto&& element : r) {
        std::cout << element << " ";
    }
    std::cout << std::endl;
}

template<std::ranges::forward_range Range>
void process_forward_range(Range&& r) {
    auto it = std::ranges::begin(r);
    const auto end = std::ranges::end(r);
    while (it != end) {
        std::cout << *it << " ";
        ++it;
    }
    std::cout << std::endl;
}

template<std::ranges::random_access_range Range>
void process_random_access_range(Range&& r) {
    const auto size = std::ranges::size(r);
    for (std::size_t i = 0; i < size; ++i) {
        std::cout << r[i] << " ";
    }
    std::cout << std::endl;
}
```

## Error Message Improvements

### Before and After Concepts

```cpp
// Without concepts - SFINAE approach
template<typename T>
std::enable_if_t<std::is_arithmetic_v<T>, T> 
add_without_concepts(T a, T b) {
    return a + b;
}

// With concepts - much clearer
template<std::arithmetic T>
T add_with_concepts(T a, T b) {
    return a + b;
}

// Usage
add_with_concepts(1, 2);      // OK
// add_with_concepts("a", "b"); // Clear error message: "T does not satisfy arithmetic"
```

### Custom Error Messages

```cpp
// Using concepts for better error reporting
template<typename T>
concept ValidForCalculation = std::arithmetic_v<T> || 
                             requires(T t) { 
                                 { t + t } -> std::convertible_to<T>; 
                             };

template<ValidForCalculation T>
T calculate(T a, T b) {
    return a + b;
}

// More specific concept with clear requirements
template<typename Container>
concept ReservableContainer = requires(Container c) {
    c.reserve(std::size_t{});
    requires std::same_as<decltype(c.reserve(std::size_t{})), void>;
};

template<ReservableContainer Container>
void optimized_insert(Container& c, std::size_t count) {
    c.reserve(c.size() + count);  // This will work because of the concept
    // Insert elements...
}
```

## Advanced Techniques

### Constraint Auto Parameters

```cpp
// Abbreviated function templates with concepts
void process_container(std::ranges::range auto&& container) {
    for (auto&& element : container) {
        std::cout << element << " ";
    }
    std::cout << std::endl;
}

void process_arithmetic(std::arithmetic auto value) {
    std::cout << "Processing arithmetic value: " << value << std::endl;
}

// Constraint on member functions
class ConceptualClass {
public:
    void method(std::integral auto value) {
        std::cout << "Integral value: " << value << std::endl;
    }
    
    void method(std::floating_point auto value) {
        std::cout << "Floating point value: " << value << std::endl;
    }
};
```

### Conditional Compilation with Concepts

```cpp
// Concepts for conditional compilation
template<typename T>
concept HasReserve = requires(T t) {
    t.reserve(std::size_t{});
};

template<typename T>
concept HasSize = requires(const T& t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

template<typename Container>
void efficient_container_op(Container& c) {
    if constexpr (HasReserve<Container> && HasSize<Container>) {
        // Use reserve if both methods are available
        c.reserve(c.size() + 10);
    }
    
    // Add elements...
}
```

### Constraint Propagation

```cpp
// Concepts propagate through templates
template<typename T>
concept ValidType = std::copyable<T> && std::equality_comparable<T>;

template<ValidType T>
class ValidContainer {
private:
    std::vector<T> data_;
    
public:
    void add(const T& item) {
        // T is guaranteed to be copyable here
        data_.push_back(item);
    }
    
    bool contains(const T& item) const {
        // T is guaranteed to be equality comparable here
        return std::find(data_.begin(), data_.end(), item) != data_.end();
    }
};
```

## Best Practices

### Concept Design Guidelines

1. **Start with Simple Concepts** - Begin with basic type requirements
2. **Use Standard Library Concepts** - Leverage existing concepts when possible
3. **Name Concepts Clearly** - Use descriptive names that indicate requirements
4. **Provide Adequate Testing** - Test concepts with various types
5. **Consider Performance** - Concepts are compile-time, but can affect compilation time
6. **Document Requirements** - Explain what the concept requires and why
7. **Use Refinement** - Build complex concepts from simpler ones
8. **Balance Specificity** - Make concepts specific enough to be useful but general enough to be reusable

### Migration from SFINAE

```cpp
// Old SFINAE approach
template<typename T>
auto old_method(T&& t) -> std::enable_if_t<std::is_integral_v<std::decay_t<T>>, int> {
    return static_cast<int>(t);
}

template<typename T>
auto old_method(T&& t) -> std::enable_if_t<!std::is_integral_v<std::decay_t<T>>, double> {
    return static_cast<double>(t);
}

// New concept-based approach
template<std::integral T>
int new_method(T&& t) {
    return static_cast<int>(t);
}

template<typename T>
requires (!std::integral<T>)
double new_method(T&& t) {
    return static_cast<double>(t);
}
```

Concepts and constraints provide a modern, clean approach to specifying template requirements, leading to better error messages, more maintainable code, and clearer API design.