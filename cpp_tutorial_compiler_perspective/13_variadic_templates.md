# Variadic Templates in C++

## Introduction

Variadic templates are a powerful C++ feature introduced in C++11 that allow templates to accept a variable number of arguments of different types. This feature is fundamental to implementing many of the modern C++ STL components and generic programming techniques.

## Basic Syntax

### Variadic Class Templates

```cpp
template<typename... Args>
class Tuple {
    // Args is a parameter pack containing zero or more types
};
```

### Variadic Function Templates

```cpp
template<typename... Args>
void print(Args... args) {
    // args is a parameter pack containing zero or more values
}
```

## Parameter Packs

### Pack Expansion

Parameter packs can be expanded using the `...` operator:

```cpp
template<typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << std::endl;  // C++17 fold expression
}
```

### Pack Declaration

- **Type parameter pack**: `typename... Args` or `class... Args`
- **Non-type parameter pack**: `int... Args`
- **Template parameter pack**: `template<typename> class... Args`

## Recursive Template Pattern

Before C++17 fold expressions, variadic templates were typically implemented using recursion:

```cpp
// Base case
void print() {
    // End of recursion
}

// Recursive case
template<typename T, typename... Args>
void print(T&& first, Args&&... rest) {
    std::cout << first << " ";
    print(std::forward<Args>(rest)...);
}
```

## Modern C++17 Fold Expressions

Fold expressions simplify variadic template usage:

```cpp
template<typename... Args>
auto sum(Args... args) {
    return (args + ... + 0);  // Addition fold
}

template<typename... Args>
void print_all(Args... args) {
    ((std::cout << args << " "), ...);  // Expression fold
}
```

### Types of Fold Expressions

1. **Unary left fold**: `(pack op ...)`
2. **Unary right fold**: `(... op pack)`
3. **Binary left fold**: `(pack op ... op init)`
4. **Binary right fold**: `(init op ... op pack)`

## Advanced Patterns

### Perfect Forwarding with Variadic Templates

```cpp
template<typename... Args>
auto make_unique(Args&&... args) {
    return std::make_unique<std::decay_t<Args>...>(
        std::forward<Args>(args)...
    );
}
```

### Expanding Non-Type Parameter Packs

```cpp
template<int... Indices>
struct IndexSequence {};

template<int N, int... Indices>
struct MakeIndexSequence : MakeIndexSequence<N-1, N-1, Indices...> {};

template<int... Indices>
struct MakeIndexSequence<0, Indices...> {
    using type = IndexSequence<Indices...>;
};
```

### SFINAE with Variadic Templates

```cpp
template<typename... Args>
auto has_size(int) -> decltype(std::declval<Args>().size()..., std::true_type{});

template<typename... Args>
std::false_type has_size(...);
```

## Compiler Implementation

### Template Instantiation

When the compiler encounters a variadic template, it:

1. Parses the parameter pack
2. For each usage, creates a specialized template instantiation
3. Expands the pack according to the specified pattern

### Code Generation

The compiler generates code for each unique combination of template arguments:

```cpp
// These will generate different instantiations:
make_tuple(1, 2.0, "hello");      // Tuple<int, double, const char*>
make_tuple(42, 3.14);             // Tuple<int, double>
```

## STL Implementation Uses

### std::tuple

```cpp
template<typename... Types>
class tuple {
    // Uses recursive inheritance pattern
    template<size_t I, typename T>
    struct TupleElement;

    // Implementation details using parameter pack expansion
};
```

### std::function

```cpp
template<typename R, typename... Args>
class function<R(Args...)> {
    // Stores callable objects with variable argument types
};
```

### Container Emplace Methods

```cpp
template<typename... Args>
iterator emplace(const_iterator pos, Args&&... args);
```

## Performance Considerations

### Compile Time

Variadic templates can significantly increase compile time, especially with deep recursion or complex pack expansion operations.

### Runtime

When properly implemented, variadic templates have zero runtime overhead compared to non-variadic alternatives.

## Common Patterns

### The "Sink" Pattern

```cpp
template<typename... Args>
void sink(Args... args) {
    // Process all arguments in a single function
    process(std::forward<Args>(args)...);
}
```

### Counting Pack Elements

```cpp
template<typename... Args>
constexpr size_t count = sizeof...(Args);
```

### Index-based Access

```cpp
template<size_t I, typename Head, typename... Tail>
struct type_at_index<I, Head, Tail...> {
    using type = typename type_at_index<I-1, Tail...>::type;
};

template<typename Head, typename... Tail>
struct type_at_index<0, Head, Tail...> {
    using type = Head;
};
```

## Advanced Techniques

### Variadic Inheritance

```cpp
template<typename... Mixins>
class Base : public Mixins... {
    // Inherit from all provided mixin classes
};
```

### Template Parameter Pack Expansion in Different Contexts

```cpp
template<typename... Args>
class VariadicExample {
    std::tuple<Args...> data;                 // Expands types
    std::function<void(Args...)> func;        // Expands function parameters
    std::array<int, sizeof...(Args)> sizes;   // Expands to integral constant
};
```

## Real-World Applications

Variadic templates are used extensively in:

- Logging frameworks
- Serialization libraries
- Testing frameworks
- Generic container implementations
- Metaprogramming libraries

Understanding variadic templates is crucial for implementing efficient and flexible generic components in C++.