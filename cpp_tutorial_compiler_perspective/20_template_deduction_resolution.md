# Template Deduction and Resolution

## Introduction

Template deduction and resolution are fundamental mechanisms in C++ that determine how template arguments are inferred from function calls and how overloaded templates are selected. Understanding these mechanisms is crucial for writing effective generic code and comprehending how the STL works under the hood.

## Template Argument Deduction

### Basic Template Deduction

```cpp
// Function template with type deduction
template<typename T>
void func(T param) {
    // T is deduced from the argument type
}

// Usage examples
func(42);           // T deduced as int
func(3.14);         // T deduced as double
func("hello");      // T deduced as const char*

// Reference parameters
template<typename T>
void func_ref(T& param) {
    // T is deduced without reference
}

int x = 42;
func_ref(x);        // T deduced as int, param type is int&

// Universal references
template<typename T>
void func_universal(T&& param) {
    // T&& is a universal reference
}

func_universal(42);     // T deduced as int, param type is int&&
func_universal(x);      // T deduced as int&, param type is int&

// Pointer parameters
template<typename T>
void func_ptr(T* param) {
    // T is deduced without pointer
}

int* ptr = &x;
func_ptr(&x);       // T deduced as int, param type is int*
func_ptr(ptr);      // T deduced as int, param type is int*
```

### Array and Function Decay

```cpp
// Array parameters
template<typename T>
void func_array(T param) {
    // Arrays decay to pointers
}

int arr[10];
func_array(arr);    // T deduced as int*, not int[10]

// Reference to array
template<typename T>
void func_array_ref(T& param) {
    // No decay with reference
}

func_array_ref(arr); // T deduced as int[10], param type is int(&)[10]

// Function parameters
void some_function(int, double);

template<typename T>
void func_func(T param) {
    // Functions decay to function pointers
}

func_func(some_function); // T deduced as void(*)(int, double)

// Reference to function
template<typename T>
void func_func_ref(T& param) {
    // No decay with reference
}

func_func_ref(some_function); // T deduced as void(int, double)
```

### Auto Type Deduction

```cpp
// Auto follows the same rules as template deduction
auto x = 27;        // auto deduced as int
const auto cx = x;  // auto deduced as int
auto& rx = x;       // auto deduced as int

// Auto with braced initializers
auto y = {1, 2, 3}; // auto deduced as std::initializer_list<int>
auto z{1, 2, 3};    // auto deduced as std::initializer_list<int> (C++17)

// Auto with universal references
template<typename Container>
void process_container(Container&& c) {
    // Using auto with universal reference
    for (auto&& element : c) {
        // element is a universal reference
    }
}
```

## Deduction Rules

### Reference Collapsing

```cpp
// Reference collapsing rules
template<typename T>
void demonstrate_collapsing() {
    // T& &  -> T&
    // T& && -> T&
    // T&& & -> T&
    // T&& && -> T&&
}

// Practical example
template<typename T>
void forward_example(T&& param) {
    // When T is int, param is int&&
    // When T is int&, param is int& (due to collapsing)
}

int x = 42;
forward_example(42);  // T = int, param = int&&
forward_example(x);   // T = int&, param = int& (collapsed from int& &&)
```

### Template Argument Deduction for Class Templates (C++17)

```cpp
// Class template argument deduction
template<typename T>
class MyVector {
public:
    MyVector(std::initializer_list<T> init) {
        // Constructor for deduction
    }
    
    template<typename Iterator>
    MyVector(Iterator first, Iterator last) {
        // Another constructor for deduction
    }
};

// Deduction guides (implicit or explicit)
MyVector v1{1, 2, 3};           // T deduced as int
std::vector v2{1, 2, 3};        // T deduced as int (STL example)

// Explicit deduction guides
template<typename Iterator>
MyVector(Iterator, Iterator) -> MyVector<typename std::iterator_traits<Iterator>::value_type>;

std::vector<int> vec = {1, 2, 3};
MyVector v3(vec.begin(), vec.end()); // T deduced as int via deduction guide
```

## Overload Resolution

### Function Template Overloading

```cpp
// Overloaded function templates
template<typename T>
void process(T value) {
    std::cout << "Generic template" << std::endl;
}

template<typename T>
void process(T* value) {
    std::cout << "Pointer template" << std::endl;
}

template<>
void process<>(int value) {
    std::cout << "Explicit specialization for int" << std::endl;
}

// Usage
int x = 42;
process(x);     // Calls process<int>(T value)
process(&x);    // Calls process<int*>(T* value)
```

### Partial Ordering of Function Templates

```cpp
// Partial ordering determines which template is more specialized
template<typename T>
void func(T) {          // Template 1
    std::cout << "Template 1" << std::endl;
}

template<typename T>
void func(T*) {         // Template 2
    std::cout << "Template 2" << std::endl;
}

int x = 42;
func(x);    // Calls Template 1 (T = int)
func(&x);   // Calls Template 2 (T = int) - more specialized

// More complex example
template<typename T>
void complex_func(T, T) {   // Template A
    std::cout << "Template A" << std::endl;
}

template<typename T>
void complex_func(T*, T*) { // Template B
    std::cout << "Template B" << std::endl;
}

int a = 1, b = 2;
int* pa = &a, *pb = &b;

complex_func(a, b);     // Calls Template A
complex_func(pa, pb);   // Calls Template B - more specialized
complex_func(a, pa);    // Ambiguous - error
```

### SFINAE and Overload Resolution

```cpp
// SFINAE (Substitution Failure Is Not An Error)
template<typename T>
void sfinae_func(T value, 
                 typename std::enable_if<std::is_integral<T>::value>::type* = nullptr) {
    std::cout << "Integral version" << std::endl;
}

template<typename T>
void sfinae_func(T value,
                 typename std::enable_if<std::is_floating_point<T>::value>::type* = nullptr) {
    std::cout << "Floating point version" << std::endl;
}

// Usage
sfinae_func(42);        // Calls integral version
sfinae_func(3.14);      // Calls floating point version
// sfinae_func("hello");   // No matching function - substitution failure

// Modern SFINAE with void_t (C++17)
template<typename, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> : std::true_type {};

template<typename T>
void size_func(T& container, std::enable_if_t<has_size<T>::value>* = nullptr) {
    std::cout << "Container has size: " << container.size() << std::endl;
}

template<typename T>
void size_func(T& container, std::enable_if_t<!has_size<T>::value>* = nullptr) {
    std::cout << "Container has no size method" << std::endl;
}
```

## Modern C++ Deduction Features

### Concepts and Constraints (C++20)

```cpp
// Concepts provide better error messages and overload resolution
template<typename T>
concept Integral = std::is_integral_v<T>;

template<typename T>
concept FloatingPoint = std::is_floating_point_v<T>;

template<Integral T>
void modern_func(T value) {
    std::cout << "Integral concept version" << std::endl;
}

template<FloatingPoint T>
void modern_func(T value) {
    std::cout << "Floating point concept version" << std::endl;
}

// Usage
modern_func(42);    // Calls Integral version
modern_func(3.14);  // Calls FloatingPoint version
// modern_func("hello"); // Compilation error with clear message
```

### Abbreviated Function Templates (C++20)

```cpp
// Abbreviated function templates
void abbreviated_func(Integral auto value) {
    std::cout << "Abbreviated integral function" << std::endl;
}

void abbreviated_func(FloatingPoint auto value) {
    std::cout << "Abbreviated floating point function" << std::endl;
}

// Equivalent to:
// template<Integral T>
// void abbreviated_func(T value) { ... }
```

### Template Parameter Lists

```cpp
// Template template parameters
template<template<typename> class Container, typename T>
class Wrapper {
    Container<T> data;
public:
    void add(const T& item) { data.push_back(item); }
};

// Usage
Wrapper<std::vector, int> vec_wrapper;
Wrapper<std::list, std::string> list_wrapper;

// Non-type template parameters with auto (C++17)
template<auto Value>
struct Constant {
    static constexpr auto value = Value;
};

Constant<42> int_constant;      // Value is int
Constant<3.14> double_constant; // Value is double
```

## STL Implementation Examples

### std::vector Deduction Guides

```cpp
// How std::vector implements deduction guides
namespace std {
    template<typename T, typename Allocator = std::allocator<T>>
    class vector {
    public:
        // Constructors for deduction
        vector(std::initializer_list<T> init, 
               const Allocator& alloc = Allocator());
        
        template<typename InputIterator>
        vector(InputIterator first, InputIterator last,
               const Allocator& alloc = Allocator());
    };
    
    // Deduction guides
    template<typename T>
    vector(std::initializer_list<T>) -> vector<T>;
    
    template<typename InputIterator>
    vector(InputIterator, InputIterator) -> 
        vector<typename std::iterator_traits<InputIterator>::value_type>;
}

// Usage examples
std::vector v1{1, 2, 3};        // Deduces vector<int>
std::vector v2(v1.begin(), v1.end()); // Deduces vector<int>
```

### std::pair Deduction

```cpp
// std::make_pair vs. deduction
std::pair<int, double> p1(42, 3.14);        // Explicit
auto p2 = std::make_pair(42, 3.14);         // Deduced as pair<int, double>
std::pair p3(42, 3.14);                     // C++17 deduction

// Custom pair-like class with deduction
template<typename T1, typename T2>
class MyPair {
public:
    T1 first;
    T2 second;
    
    MyPair(T1&& f, T2&& s) : first(std::forward<T1>(f)), second(std::forward<T2>(s)) {}
};

// Deduction guide
template<typename T1, typename T2>
MyPair(T1, T2) -> MyPair<T1, T2>;

// Usage
MyPair mp(42, 3.14); // Deduces MyPair<int, double>
```

## Complex Deduction Scenarios

### Nested Template Deduction

```cpp
// Complex deduction with nested templates
template<typename T>
class Outer {
public:
    template<typename U>
    class Inner {
    public:
        void func(T t, U u) {
            // T comes from Outer, U comes from Inner
        }
    };
};

// Usage
Outer<int>::Inner<double> inner;
inner.func(42, 3.14); // T = int, U = double

// Deduction with nested function templates
template<typename T>
void outer_func() {
    // Nested function template
    auto lambda = []<typename U>(U value) {
        // U is deduced from lambda call
        std::cout << value << std::endl;
    };
    
    lambda(42);     // U deduced as int
    lambda(3.14);   // U deduced as double
}
```

### Variadic Template Deduction

```cpp
// Variadic template deduction
template<typename... Args>
void variadic_func(Args... args) {
    // Args are deduced from the argument list
    std::cout << "Number of arguments: " << sizeof...(Args) << std::endl;
}

// Usage
variadic_func(1, 2.0, "three"); // Args deduced as int, double, const char*

// Perfect forwarding with variadic templates
template<typename... Args>
void forward_variadic(Args&&... args) {
    // Args&& are universal references
    another_function(std::forward<Args>(args)...);
}
```

## Troubleshooting Deduction Issues

### Common Deduction Problems

```cpp
// Problem: Braced initializers with auto
auto x{42};     // C++17: x is int
auto y = {42};  // y is std::initializer_list<int>

// Problem: Reference collapsing confusion
template<typename T>
void problematic_func(T&& param) {
    // param is universal reference, not rvalue reference
}

// Solution: Understanding when universal references apply
template<typename T>
class Container {
public:
    void push_back(T&& item);           // rvalue reference (T is known)
    template<typename U>
    void emplace_back(U&& item);        // universal reference (U is deduced)
};

// Problem: Template argument deduction failure
template<typename T>
void deduction_failure(T* ptr, T value) {
    // T must match exactly - no implicit conversions in deduction
}

int x = 42;
// deduction_failure(&x, 3.14); // Error: T cannot be both int and double

// Solution: Separate template parameters
template<typename T1, typename T2>
void separate_params(T1* ptr, T2 value) {
    // Now T1 and T2 can be different types
}
```

### Debugging Deduction

```cpp
// Debugging template deduction
template<typename T>
void debug_deduction(T&& param) {
    std::cout << "T is: " << typeid(T).name() << std::endl;
    std::cout << "param type is: " << typeid(decltype(param)).name() << std::endl;
}

// Using static_assert for compile-time checking
template<typename T>
void checked_func(T param) {
    static_assert(std::is_integral_v<T>, "T must be integral");
    // Function body
}

// Type traits for deduction verification
template<typename T>
void verify_deduction() {
    std::cout << "T is integral: " << std::is_integral_v<T> << std::endl;
    std::cout << "T is reference: " << std::is_reference_v<T> << std::endl;
    std::cout << "T is const: " << std::is_const_v<std::remove_reference_t<T>> << std::endl;
}
```

## Best Practices

### Template Deduction Guidelines

1. **Understand Reference Collapsing** - Know how universal references work
2. **Use auto Carefully** - Be aware of deduction differences with braced initializers
3. **Prefer Concepts** - Use C++20 concepts for better error messages
4. **Provide Deduction Guides** - Help users with class template argument deduction
5. **Avoid Over-Specialization** - Let deduction work naturally when possible
6. **Test Edge Cases** - Verify deduction works with various argument types
7. **Document Deduction Behavior** - Explain how templates are expected to deduce

Understanding template deduction and resolution is essential for writing effective generic C++ code and for comprehending how modern C++ libraries are designed.