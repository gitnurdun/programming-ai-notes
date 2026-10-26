# Advanced Template Metaprogramming

## Introduction

Advanced template metaprogramming builds upon basic techniques to implement sophisticated compile-time algorithms and data structures. These techniques are crucial for understanding complex STL implementations and for developing advanced C++ libraries.

## Template Template Parameters

Template template parameters allow passing templates as parameters to other templates:

```cpp
template<template<typename> class Container, typename T>
class GenericWrapper {
    Container<T> data;
public:
    void add(const T& item) { data.push_back(item); }
    size_t size() const { return data.size(); }
};

// Usage
GenericWrapper<std::vector, int> vec_wrapper;
GenericWrapper<std::list, std::string> list_wrapper;
```

### Complex Template Template Parameters

```cpp
template<template<typename, typename> class Container, 
         typename T, 
         typename Allocator = std::allocator<T>>
class ComplexWrapper {
    Container<T, Allocator> container;
public:
    using value_type = T;
    void push(const T& item) { container.push_back(item); }
};
```

## Template Recursion with Type Lists

### Type List Implementation

```cpp
template<typename... Ts>
struct TypeList {};

// Getting the first type
template<typename List>
struct front;

template<typename Head, typename... Tail>
struct front<TypeList<Head, Tail...>> {
    using type = Head;
};

// Getting the rest of the list
template<typename List>
struct pop_front;

template<typename Head, typename... Tail>
struct pop_front<TypeList<Head, Tail...>> {
    using type = TypeList<Tail...>;
};

// Adding to the front
template<typename T, typename List>
struct push_front;

template<typename T, typename... Ts>
struct push_front<T, TypeList<Ts...>> {
    using type = TypeList<T, Ts...>;
};
```

## Template Metaprogramming Algorithms

### Compile-time List Operations

```cpp
// Length calculation
template<typename List>
struct length;

template<typename... Ts>
struct length<TypeList<Ts...>> {
    static constexpr size_t value = sizeof...(Ts);
};

// Finding an element
template<typename List, typename T>
struct contains;

template<typename T>
struct contains<TypeList<>, T> : std::false_type {};

template<typename Head, typename... Tail, typename T>
struct contains<TypeList<Head, Tail...>, T> 
    : std::conditional_t<std::is_same_v<Head, T>, 
                        std::true_type, 
                        contains<TypeList<Tail...>, T>> {};

// List concatenation
template<typename List1, typename List2>
struct concat;

template<typename... Ts1, typename... Ts2>
struct concat<TypeList<Ts1...>, TypeList<Ts2...>> {
    using type = TypeList<Ts1..., Ts2...>;
};
```

### Transform Operations

```cpp
// Apply transformation to each element
template<typename List, template<typename> class Transform>
struct transform;

template<template<typename> class Transform, typename... Ts>
struct transform<TypeList<Ts...>, Transform> {
    using type = TypeList<typename Transform<Ts>::type...>;
};

// Example transformation
template<typename T>
struct add_pointer {
    using type = T*;
};

// Usage: transform<TypeList<int, double>, add_pointer>::type
// Results in TypeList<int*, double*>
```

## Advanced SFINAE Techniques

### Expression SFINAE (C++11)

```cpp
template<typename T>
constexpr bool has_begin_method(int) {
    // Check if T has a begin() method
    return std::is_same_v<
        decltype(std::declval<T>().begin()),
        typename T::iterator
    >;
}

template<typename T>
constexpr bool has_begin_method(long) {
    return false;
}

template<typename T>
struct has_begin {
    static constexpr bool value = has_begin_method<T>(int{});
};
```

### Modern C++17 Detection Idiom

```cpp
template<typename...>
using void_t = void;

template<typename, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, void_t<decltype(std::declval<T>().size())>> 
    : std::true_type {};
```

### C++20 Concepts Alternative

```cpp
template<typename T>
concept HasSize = requires(T t) {
    t.size();
};

template<HasSize T>
void process_container(const T& container);
```

## Variadic Template Metaprogramming

### Index Sequence Magic

```cpp
template<std::size_t... Is>
struct index_sequence {};

template<std::size_t N, std::size_t... Is>
struct make_index_sequence 
    : make_index_sequence<N-1, N-1, Is...> {};

template<std::size_t... Is>
struct make_index_sequence<0, Is...> {
    using type = index_sequence<Is...>;
};

// Usage for expanding parameter packs
template<typename Tuple, std::size_t... Is>
auto tuple_to_array_impl(Tuple&& t, index_sequence<Is...>) {
    return std::array{std::get<Is>(std::forward<Tuple>(t))...};
}

template<typename... Ts>
auto tuple_to_array(const std::tuple<Ts...>& t) {
    return tuple_to_array_impl(t, make_index_sequence<sizeof...(Ts)>{});
}
```

## Metaclass Programming

### Type Inspection

```cpp
template<typename T>
struct type_info {
    static constexpr bool is_arithmetic = std::is_arithmetic_v<T>;
    static constexpr bool is_class = std::is_class_v<T>;
    static constexpr bool is_enum = std::is_enum_v<T>;
    static constexpr size_t alignment = alignof(T);
    static constexpr size_t size = sizeof(T);
};
```

### Member Function Detection

```cpp
template<typename T>
struct has_member_functions {
    template<typename U>
    static auto has_begin_test(int) -> decltype(
        std::declval<U>().begin(), 
        std::true_type{}
    );
    
    template<typename U>
    static std::false_type has_begin_test(...);
    
    static constexpr bool has_begin = 
        decltype(has_begin_test<T>(0))::value;
};
```

## Compile-time Reflection (Before C++26)

### Enum to String

```cpp
enum class Color { Red, Green, Blue };

template<Color C>
constexpr const char* color_to_string() {
    if constexpr (C == Color::Red) return "Red";
    else if constexpr (C == Color::Green) return "Green";
    else if constexpr (C == Color::Blue) return "Blue";
    else return "Unknown";
}
```

## Advanced Type Traits

### Type Modification Traits

```cpp
// Remove all cv-qualifiers and references
template<typename T>
struct remove_cvref {
    using type = std::remove_cv_t<std::remove_reference_t<T>>;
};

template<typename T>
using remove_cvref_t = typename remove_cvref<T>::type;

// Conditional type modification
template<typename T>
struct decay_if_pointer {
    using type = std::conditional_t<
        std::is_pointer_v<T>,
        std::remove_pointer_t<T>,
        T
    >;
};
```

### Function Type Traits

```cpp
// Extract function signature components
template<typename F>
struct function_traits;

template<typename R, typename... Args>
struct function_traits<R(Args...)> {
    using return_type = R;
    using arguments = TypeList<Args...>;
    static constexpr size_t arity = sizeof...(Args);
    
    template<size_t I>
    struct arg {
        using type = typename std::tuple_element<I, std::tuple<Args...>>::type;
    };
};
```

## Modern C++ Approaches

### Concepts and Constraints (C++20)

```cpp
template<typename T>
concept Iterable = requires(T t) {
    t.begin();
    t.end();
    { t.begin() != t.end() } -> std::convertible_to<bool>;
};

template<Iterable T>
void process_iterable(T& container) {
    // Implementation
}
```

### Template Syntax Improvements

```cpp
// Abbreviated function templates (C++20)
void sort(Iterable auto& container) {
    // Implementation
}

// Constraint auto
template<Iterable Container>
void process(Container auto& c) {
    // Implementation
}
```

## Performance Considerations

### Compile Time vs Runtime Trade-offs

Template metaprograms can significantly increase compilation time but provide runtime efficiency:

- Complex metaprograms may require more compiler resources
- Generated code is often more optimized than runtime alternatives
- Template instantiation depth limits exist (typically 1024 levels)

### Optimization Techniques

```cpp
// Memoization to avoid redundant instantiations
template<typename T>
struct computed_result {
    static constexpr auto value = complex_computation<T>();
};

// Use alias to access the computed value
template<typename T>
constexpr auto computed_result_v = computed_result<T>::value;
```

## Real-World STL Examples

### std::tuple Implementation

```cpp
// Simplified tuple implementation using template metaprogramming
template<typename... Types>
class tuple {
    // Use inheritance to optimize empty base class
    template<size_t I, typename T>
    struct tuple_element_impl : T {
        tuple_element_impl(T&& t) : T(std::forward<T>(t)) {}
    };
    
    // Combine all elements using inheritance
    using storage = std::tuple<tuple_element_impl<0, Types>...>;
    storage data;
};
```

### std::variant Implementation

```cpp
template<typename... Types>
class variant {
    // Uses template metaprogramming to:
    // - Determine the maximum size needed
    // - Calculate alignment requirements
    // - Implement visitation algorithms
};
```

Advanced template metaprogramming techniques provide the foundation for implementing sophisticated C++ libraries and understanding the inner workings of the STL.