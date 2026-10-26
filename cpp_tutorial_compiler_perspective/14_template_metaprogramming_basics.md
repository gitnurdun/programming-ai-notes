# Template Metaprogramming Basics

## Introduction

Template metaprogramming is a programming technique in C++ where templates are used to generate code at compile time. This technique allows for computation and decision-making at compile time, leading to efficient, type-safe code. Understanding template metaprogramming is crucial for comprehending how STL components work internally.

## Core Concepts

### What is Template Metaprogramming?

Template metaprogramming is a form of compile-time computation where the C++ template system is used to perform operations during compilation. The "program" consists of template definitions, and the "input" is the template arguments. The "output" is the generated code.

### The Template Metaprogramming "Machine"

The C++ compiler acts as a functional programming interpreter during template compilation:
- **Templates** are functions
- **Template arguments** are parameters
- **Specializations** are conditional branches
- **Instantiation** is function evaluation

## Basic Techniques

### Template Specialization

Template specialization allows defining different behavior for specific types:

```cpp
// Primary template
template<typename T>
struct is_integer {
    static const bool value = false;
};

// Specialization for int
template<>
struct is_integer<int> {
    static const bool value = true;
};

// Specialization for char
template<>
struct is_integer<char> {
    static const bool value = true;
};
```

### Partial Template Specialization

Allows specialization based on template parameters:

```cpp
// Primary template - pointer to any type
template<typename T>
struct remove_pointer {
    typedef T type;
};

// Partial specialization - pointer to const type
template<typename T>
struct remove_pointer<T*> {
    typedef T type;
};
```

### Recursive Templates

Metaprogramming often uses recursion to perform iteration at compile time:

```cpp
// Compile-time factorial calculation
template<int N>
struct factorial {
    static const int value = N * factorial<N-1>::value;
};

// Base case specialization
template<>
struct factorial<0> {
    static const int value = 1;
};
```

## Compile-time Constants

### Using constexpr (C++11)

```cpp
template<int N>
struct factorial {
    static constexpr int value = N * factorial<N-1>::value;
};

template<>
struct factorial<0> {
    static constexpr int value = 1;
};
```

### Template Value Parameters

```cpp
template<typename T, T value>
struct integral_constant {
    static constexpr T v = value;
    using value_type = T;
    using type = integral_constant<T, value>;
    
    constexpr operator value_type() const noexcept { return v; }
    constexpr value_type operator()() const noexcept { return v; }
};
```

## Type Traits

### Conditional Types

```cpp
template<bool B, typename T, typename F>
struct conditional {
    using type = T;
};

template<typename T, typename F>
struct conditional<false, T, F> {
    using type = F;
};

// Usage
using result = conditional<true, int, double>::type; // int
```

### Type Relationships

```cpp
template<typename T, typename U>
struct is_same {
    static const bool value = false;
};

template<typename T>
struct is_same<T, T> {
    static const bool value = true;
};
```

## Modern C++ Approaches

### Type Aliases (C++11)

```cpp
template<bool B, typename T, typename F>
using conditional_t = typename conditional<B, T, F>::type;

template<typename T>
using remove_pointer_t = typename remove_pointer<T>::type;
```

### Variable Templates (C++14)

```cpp
template<typename T, typename U>
constexpr bool is_same_v = is_same<T, U>::value;

template<int N>
constexpr int factorial_v = factorial<N>::value;
```

## Advanced Patterns

### SFINAE with Templates

Substitution Failure Is Not An Error allows for compile-time selection:

```cpp
template<typename T>
auto has_size(int) -> decltype(std::declval<T>().size(), std::true_type{});

template<typename T>
std::false_type has_size(...);

template<typename T>
struct has_size_trait {
    using type = decltype(has_size<T>(0));
    static constexpr bool value = type::value;
};
```

### Enable_if Pattern

Allows conditionally enabling templates:

```cpp
template<typename T>
typename std::enable_if<std::is_arithmetic<T>::value, T>::type
add(T a, T b) {
    return a + b;
}
```

### Modern Concepts Alternative (C++20)

```cpp
template<typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template<Arithmetic T>
T add(T a, T b) {
    return a + b;
}
```

## Compiler Implementation Details

### Template Instantiation Process

1. **Parsing**: Compiler parses template definitions
2. **Name Binding**: Identifies template parameters and their usage
3. **Substitution**: Replaces template parameters with actual arguments
4. **Type Checking**: Validates the substituted template
5. **Code Generation**: Generates the final code

### Template Deduction

The compiler performs template argument deduction:
```cpp
template<typename T>
void func(T param);  // T is deduced from the argument

func(42);  // T is deduced as int
```

## Practical Applications in STL

### std::enable_if Implementation

```cpp
template<bool B, class T = void>
struct enable_if {};

template<class T>
struct enable_if<true, T> { typedef T type; };
```

### Type Traits in STL

STL uses template metaprogramming for type introspection:

```cpp
// Used internally in containers to determine if types are move-constructible
template<typename T>
struct is_move_constructible {
    // Implementation using SFINAE and other metaprogramming techniques
};
```

### Compile-time Size Calculations

```cpp
// Used in std::tuple implementation
template<typename... Types>
struct tuple_size;

template<typename... Types>
constexpr size_t tuple_size_v = tuple_size<Types...>::value;
```

## Performance Benefits

### Zero Runtime Overhead

Template metaprograms execute at compile time, resulting in no runtime overhead.

### Optimized Code Generation

The compiler can optimize based on compile-time computed values, often producing more efficient code than runtime alternatives.

## Common Patterns

### The Expression Template Pattern

Used in numeric libraries to optimize expressions:

```cpp
template<typename L, typename R>
class AddExpression {
    const L& left;
    const R& right;
public:
    AddExpression(const L& l, const R& r) : left(l), right(r) {}
    
    double operator[](size_t i) const {
        return left[i] + right[i];
    }
};
```

### Policy-Based Design

Allows different implementation strategies:

```cpp
template<typename AllocationPolicy, typename ThreadingPolicy>
class Container {
    // Implementation using the provided policies
};
```

Template metaprogramming is fundamental to modern C++ and STL implementation, enabling efficient, type-safe, and flexible code generation at compile time.