# Template Specialization and Instantiation

## Introduction

Template specialization and instantiation are fundamental concepts in C++ template programming that allow developers to provide custom implementations for specific types or conditions. Understanding these concepts is essential for creating efficient, type-safe generic code and for comprehending how the STL provides specialized implementations for different types.

## Template Instantiation

### Implicit Instantiation

```cpp
// Implicit instantiation occurs when the compiler generates code for a template
template<typename T>
class SimpleContainer {
private:
    T value_;
    
public:
    SimpleContainer(const T& value) : value_(value) {}
    
    T get() const { return value_; }
    void set(const T& value) { value_ = value; }
};

// Implicit instantiation examples
void implicit_instantiation_examples() {
    // These lines cause implicit instantiation:
    SimpleContainer<int> int_container(42);           // Instantiates SimpleContainer<int>
    SimpleContainer<double> double_container(3.14);   // Instantiates SimpleContainer<double>
    SimpleContainer<std::string> string_container("hello"); // Instantiates SimpleContainer<std::string>
    
    int x = int_container.get();        // Uses instantiated SimpleContainer<int>
    double y = double_container.get();  // Uses instantiated SimpleContainer<double>
}

// Function template implicit instantiation
template<typename T>
T max_value(T a, T b) {
    return (a > b) ? a : b;
}

void function_instantiation_examples() {
    // Implicit instantiation of function templates
    int max_int = max_value(10, 20);              // Instantiates max_value<int>
    double max_double = max_value(1.5, 2.7);      // Instantiates max_value<double>
    std::string max_string = max_value(std::string("a"), std::string("b")); // Instantiates max_value<std::string>
}
```

### Explicit Instantiation

```cpp
// Explicit instantiation - forcing the compiler to generate code for specific types
template<typename T>
class ExplicitInstantiationExample {
private:
    std::vector<T> data_;
    
public:
    void add(const T& value) { data_.push_back(value); }
    std::size_t size() const { return data_.size(); }
    const T& at(std::size_t index) const { return data_.at(index); }
};

// Explicit instantiation definitions (usually in .cpp files)
template class ExplicitInstantiationExample<int>;
template class ExplicitInstantiationExample<double>;
template class ExplicitInstantiationExample<std::string>;

// Function template explicit instantiation
template<typename T>
void process_data(const std::vector<T>& data) {
    for (const auto& item : data) {
        std::cout << item << " ";
    }
    std::cout << std::endl;
}

// Explicit instantiation of function template
template void process_data<int>(const std::vector<int>&);
template void process_data<double>(const std::vector<double>&);

// Usage in another translation unit
void use_explicitly_instantiated_templates() {
    ExplicitInstantiationExample<int> int_example;
    int_example.add(42);
    
    std::vector<int> int_data = {1, 2, 3, 4, 5};
    process_data(int_data);  // Uses explicitly instantiated version
}
```

### Instantiation Depth and Recursion

```cpp
// Template instantiation depth and recursive templates
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N-1>::value;
};

// Base case specialization
template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

// Deep instantiation example
void deep_instantiation_example() {
    // This will instantiate Factorial<5>, Factorial<4>, ..., Factorial<0>
    constexpr int result = Factorial<5>::value;  // 120
    std::cout << "5! = " << result << std::endl;
}

// Template recursion with type lists
template<typename... Types>
struct TypeList {};

template<typename List>
struct Length;

template<typename... Types>
struct Length<TypeList<Types...>> {
    static constexpr std::size_t value = sizeof...(Types);
};

template<typename List>
struct Front;

template<typename Head, typename... Tail>
struct Front<TypeList<Head, Tail...>> {
    using type = Head;
};

// Complex instantiation example
void type_list_instantiation_example() {
    using MyList = TypeList<int, double, std::string, char>;
    constexpr std::size_t length = Length<MyList>::value;  // 4
    
    using FirstType = typename Front<MyList>::type;  // int
    
    std::cout << "List length: " << length << std::endl;
    std::cout << "First type is int: " << std::is_same_v<FirstType, int> << std::endl;
}
```

## Template Specialization

### Full Template Specialization

```cpp
// Primary template
template<typename T>
class Storage {
private:
    T data_;
    
public:
    Storage(const T& value) : data_(value) {}
    
    void print() const {
        std::cout << "Generic storage: " << data_ << std::endl;
    }
    
    std::size_t size() const {
        return sizeof(T);
    }
};

// Full specialization for bool
template<>
class Storage<bool> {
private:
    bool data_;
    
public:
    Storage(bool value) : data_(value) {}
    
    void print() const {
        std::cout << "Bool storage: " << (data_ ? "true" : "false") << std::endl;
    }
    
    std::size_t size() const {
        return 1;  // Optimized size for bool
    }
};

// Full specialization for std::string
template<>
class Storage<std::string> {
private:
    std::string data_;
    
public:
    Storage(const std::string& value) : data_(value) {}
    
    void print() const {
        std::cout << "String storage: \"" << data_ << "\" (length: " << data_.length() << ")" << std::endl;
    }
    
    std::size_t size() const {
        return data_.length();
    }
};

// Full specialization for pointer types
template<typename T>
class Storage<T*> {
private:
    T* data_;
    
public:
    Storage(T* value) : data_(value) {}
    
    void print() const {
        std::cout << "Pointer storage: " << data_ << " (address: " << static_cast<void*>(data_) << ")" << std::endl;
    }
    
    std::size_t size() const {
        return sizeof(T*);
    }
};

// Usage examples
void full_specialization_examples() {
    Storage<int> int_storage(42);
    Storage<bool> bool_storage(true);
    Storage<std::string> string_storage("Hello");
    Storage<double*> ptr_storage(nullptr);
    
    int_storage.print();     // Generic storage: 42
    bool_storage.print();    // Bool storage: true
    string_storage.print();  // String storage: "Hello" (length: 5)
    ptr_storage.print();     // Pointer storage: 0 (address: 0)
}
```

### Partial Template Specialization

```cpp
// Partial specialization for class templates
template<typename T, typename Allocator>
class Container {
public:
    void info() const {
        std::cout << "Generic container with allocator" << std::endl;
    }
};

// Partial specialization for std::allocator
template<typename T>
class Container<T, std::allocator<T>> {
public:
    void info() const {
        std::cout << "Container with default allocator" << std::endl;
    }
};

// Partial specialization for pointer types
template<typename T, typename Allocator>
class Container<T*, Allocator> {
public:
    void info() const {
        std::cout << "Container of pointers" << std::endl;
    }
};

// Partial specialization for const types
template<typename T, typename Allocator>
class Container<const T, Allocator> {
public:
    void info() const {
        std::cout << "Container of const elements" << std::endl;
    }
};

// Complex partial specialization
template<typename T>
class SmartPointer {
private:
    T* ptr_;
    
public:
    explicit SmartPointer(T* p = nullptr) : ptr_(p) {}
    ~SmartPointer() { delete ptr_; }
    
    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
};

// Partial specialization for arrays
template<typename T, std::size_t N>
class SmartPointer<T[N]> {
private:
    T* ptr_;
    
public:
    explicit SmartPointer(T* p = nullptr) : ptr_(p) {}
    ~SmartPointer() { delete[] ptr_; }
    
    T& operator[](std::size_t index) { return ptr_[index]; }
    const T& operator[](std::size_t index) const { return ptr_[index]; }
};

// Usage examples
void partial_specialization_examples() {
    Container<int, std::allocator<int>> default_container;
    Container<int, MyAllocator<int>> custom_container;
    Container<int*, std::allocator<int*>> pointer_container;
    Container<const int, std::allocator<const int>> const_container;
    
    default_container.info();  // Container with default allocator
    custom_container.info();   // Generic container with allocator
    pointer_container.info();  // Container of pointers
    const_container.info();    // Container of const elements
    
    SmartPointer<int> single_ptr(new int(42));
    SmartPointer<int[10]> array_ptr(new int[10]);
    
    *single_ptr = 100;
    array_ptr[5] = 200;
}
```

### Function Template Specialization

```cpp
// Function templates can only have full specializations (not partial)
template<typename T>
void process(T value) {
    std::cout << "Generic processing: " << value << std::endl;
}

// Full specialization for int
template<>
void process<int>(int value) {
    std::cout << "Specialized processing for int: " << value << std::endl;
}

// Full specialization for std::string
template<>
void process<std::string>(std::string value) {
    std::cout << "Specialized processing for string: \"" << value << "\"" << std::endl;
}

// SFINAE-based function selection (preferred over specialization)
template<typename T>
std::enable_if_t<std::is_arithmetic_v<T>, void>
process(T value) {
    std::cout << "Arithmetic processing: " << value << std::endl;
}

template<typename T>
std::enable_if_t<!std::is_arithmetic_v<T>, void>
process(T value) {
    std::cout << "Non-arithmetic processing: " << value << std::endl;
}

// Modern C++20 concepts approach
template<std::integral T>
void modern_process(T value) {
    std::cout << "Integral processing: " << value << std::endl;
}

template<std::floating_point T>
void modern_process(T value) {
    std::cout << "Floating point processing: " << value << std::endl;
}

template<typename T>
void modern_process(T value) requires (!std::is_arithmetic_v<T>)
{
    std::cout << "Non-arithmetic processing: " << value << std::endl;
}

// Usage examples
void function_specialization_examples() {
    process(42);           // Specialized processing for int
    process(3.14);         // Generic processing: 3.14
    process(std::string("hello")); // Specialized processing for string
    
    modern_process(42);    // Integral processing: 42
    modern_process(3.14);  // Floating point processing: 3.14
    modern_process(std::vector<int>{}); // Non-arithmetic processing
}
```

## Advanced Specialization Techniques

### Specialization with Type Traits

```cpp
// Using type traits for conditional specialization
template<typename T, typename Enable = void>
class ConditionalStorage {
public:
    void store(const T& value) {
        std::cout << "Generic storage for: " << value << std::endl;
    }
};

// Specialization for integral types
template<typename T>
class ConditionalStorage<T, std::enable_if_t<std::is_integral_v<T>>> {
public:
    void store(const T& value) {
        std::cout << "Integral storage: " << value << " (0x" << std::hex << value << std::dec << ")" << std::endl;
    }
};

// Specialization for floating-point types
template<typename T>
class ConditionalStorage<T, std::enable_if_t<std::is_floating_point_v<T>>> {
public:
    void store(const T& value) {
        std::cout << "Floating-point storage: " << std::fixed << std::setprecision(2) << value << std::endl;
    }
};

// Specialization for container types
template<typename T>
class ConditionalStorage<T, std::enable_if_t<is_container_v<T>>> {
public:
    void store(const T& value) {
        std::cout << "Container storage with " << value.size() << " elements" << std::endl;
    }
};

// Type trait to detect containers
template<typename T, typename = void>
struct is_container : std::false_type {};

template<typename T>
struct is_container<T, std::void_t<
    typename T::value_type,
    typename T::size_type,
    typename T::iterator,
    decltype(std::declval<T>().size()),
    decltype(std::declval<T>().begin()),
    decltype(std::declval<T>().end())
>> : std::true_type {};

template<typename T>
constexpr bool is_container_v = is_container<T>::value;

// Usage examples
void conditional_specialization_examples() {
    ConditionalStorage<int> int_storage;
    ConditionalStorage<double> double_storage;
    ConditionalStorage<std::vector<int>> vector_storage;
    ConditionalStorage<std::string> string_storage;
    
    int_storage.store(42);        // Integral storage: 42 (0x2a)
    double_storage.store(3.14159); // Floating-point storage: 3.14
    vector_storage.store(std::vector<int>{1, 2, 3}); // Container storage with 3 elements
    string_storage.store("hello"); // Generic storage for: hello
}
```

### Template Template Specialization

```cpp
// Template template parameters and specialization
template<template<typename, typename> class Container, typename T, typename Allocator>
class ContainerWrapper {
private:
    Container<T, Allocator> container_;
    
public:
    void add(const T& value) { container_.push_back(value); }
    
    void print() const {
        std::cout << "Generic container wrapper" << std::endl;
        for (const auto& item : container_) {
            std::cout << item << " ";
        }
        std::cout << std::endl;
    }
};

// Specialization for std::vector
template<typename T, typename Allocator>
class ContainerWrapper<std::vector, T, Allocator> {
private:
    std::vector<T, Allocator> container_;
    
public:
    void add(const T& value) { container_.push_back(value); }
    
    void print() const {
        std::cout << "Vector wrapper (size: " << container_.size() << ")" << std::endl;
        for (const auto& item : container_) {
            std::cout << item << " ";
        }
        std::cout << std::endl;
    }
    
    // Vector-specific operations
    void reserve(std::size_t capacity) { container_.reserve(capacity); }
    std::size_t capacity() const { return container_.capacity(); }
};

// Specialization for std::list
template<typename T, typename Allocator>
class ContainerWrapper<std::list, T, Allocator> {
private:
    std::list<T, Allocator> container_;
    
public:
    void add(const T& value) { container_.push_back(value); }
    
    void print() const {
        std::cout << "List wrapper (size: " << container_.size() << ")" << std::endl;
        for (const auto& item : container_) {
            std::cout << item << " ";
        }
        std::cout << std::endl;
    }
    
    // List-specific operations
    void sort() { container_.sort(); }
};

// Usage examples
void template_template_specialization_examples() {
    ContainerWrapper<std::vector, int, std::allocator<int>> vector_wrapper;
    ContainerWrapper<std::list, int, std::allocator<int>> list_wrapper;
    ContainerWrapper<std::deque, int, std::allocator<int>> deque_wrapper; // Uses generic version
    
    for (int i = 5; i > 0; --i) {
        vector_wrapper.add(i);
        list_wrapper.add(i);
        deque_wrapper.add(i);
    }
    
    vector_wrapper.print();  // Vector wrapper (size: 5)
    list_wrapper.print();    // List wrapper (size: 5)
    deque_wrapper.print();   // Generic container wrapper
}
```

## Compilation and Linking Considerations

### Template Instantiation and Linkage

```cpp
// Header file: template_declaration.hpp
#ifndef TEMPLATE_DECLARATION_HPP
#define TEMPLATE_DECLARATION_HPP

template<typename T>
class TemplateWithLinkage {
public:
    void method1();
    void method2();
    
    // Inline method - implicitly instantiated where used
    void inline_method() {
        std::cout << "Inline method" << std::endl;
    }
};

// Explicit instantiation declaration (C++11)
extern template class TemplateWithLinkage<int>;
extern template class TemplateWithLinkage<double>;

#endif

// Implementation file: template_implementation.cpp
#include "template_declaration.hpp"

// Method definitions
template<typename T>
void TemplateWithLinkage<T>::method1() {
    std::cout << "Method 1 implementation" << std::endl;
}

template<typename T>
void TemplateWithLinkage<T>::method2() {
    std::cout << "Method 2 implementation" << std::endl;
}

// Explicit instantiation definitions
template class TemplateWithLinkage<int>;
template class TemplateWithLinkage<double>;

// Usage file: main.cpp
#include "template_declaration.hpp"

void usage_examples() {
    TemplateWithLinkage<int> int_instance;
    TemplateWithLinkage<double> double_instance;
    TemplateWithLinkage<std::string> string_instance; // Implicit instantiation
    
    int_instance.method1();    // Uses explicitly instantiated version
    double_instance.method2(); // Uses explicitly instantiated version
    string_instance.method1(); // Implicit instantiation
}
```

### Compilation-Time Optimization

```cpp
// Template instantiation optimization techniques
template<typename T>
class OptimizedTemplate {
private:
    T data_;
    
public:
    OptimizedTemplate(const T& value) : data_(value) {}
    
    // Constexpr methods for compile-time evaluation
    template<typename U = T>
    constexpr std::enable_if_t<std::is_arithmetic_v<U>, U>
    squared() const {
        return data_ * data_;
    }
    
    // Specialized methods for different types
    template<typename U = T>
    std::enable_if_t<std::is_same_v<U, std::string>, std::size_t>
    length() const {
        return data_.length();
    }
    
    // Conditional compilation based on type properties
    void print_info() const {
        if constexpr (std::is_arithmetic_v<T>) {
            std::cout << "Arithmetic value: " << data_ << std::endl;
        } else if constexpr (std::is_same_v<T, std::string>) {
            std::cout << "String value: \"" << data_ << "\" (length: " << data_.length() << ")" << std::endl;
        } else {
            std::cout << "Generic value: " << data_ << std::endl;
        }
    }
};

// Compile-time computation with templates
template<int N>
struct CompileTimeComputation {
    static constexpr int value = N * 2 + 1;
};

// Usage examples
void optimization_examples() {
    constexpr int compile_time_result = CompileTimeComputation<5>::value; // 11
    
    OptimizedTemplate<int> int_opt(42);
    OptimizedTemplate<std::string> string_opt("hello");
    
    constexpr int squared = int_opt.squared(); // Computed at compile time
    std::cout << "Compile-time result: " << compile_time_result << std::endl;
    std::cout << "Squared value: " << squared << std::endl;
    
    int_opt.print_info();    // Arithmetic value: 42
    string_opt.print_info(); // String value: "hello" (length: 5)
}
```

## Best Practices

### Template Specialization Guidelines

1. **Prefer SFINAE over Specialization** - For function templates, use SFINAE or concepts instead of specialization
2. **Use Explicit Instantiation Sparingly** - Only when you need to control compilation or reduce code size
3. **Be Consistent with Specializations** - Ensure all specializations provide the same interface
4. **Consider Performance Implications** - Specializations can affect compilation time and code size
5. **Document Specialization Behavior** - Clearly explain what each specialization does
6. **Test All Specializations** - Ensure specializations work correctly with your code
7. **Avoid Over-Specialization** - Only specialize when there's a genuine need for different behavior
8. **Use Type Traits for Conditional Behavior** - Instead of multiple specializations

### Common Specialization Pitfalls

```cpp
// Examples of common specialization mistakes to avoid

// 1. Inconsistent interfaces
template<typename T>
class BadSpecialization {
public:
    void common_method();
    void specialized_method();  // Only in generic version
};

template<>
class BadSpecialization<int> {
public:
    void common_method();
    // Missing specialized_method() - inconsistent interface!
};

// 2. Accidental specialization
template<typename T>
void process(T value) {
    std::cout << "Generic: " << value << std::endl;
}

// This is NOT a specialization - it's overloading!
template<typename T>
void process(T* value) {
    std::cout << "Pointer: " << value << std::endl;
}

// Correct specialization:
template<>
void process<int>(int value) {
    std::cout << "Specialized int: " << value << std::endl;
}

// 3. Order-dependent specialization
// Wrong order - compiler doesn't know about specialization yet
void wrong_order_example() {
    process(42);  // Calls generic version, not specialization!
}

template<>
void process<int>(int value) {
    std::cout << "Specialized int: " << value << std::endl;
}

// Correct order - specialization before use
template<>
void process<int>(int value);  // Forward declaration

void correct_order_example() {
    process(42);  // Calls specialized version
}

template<>
void process<int>(int value) {
    std::cout << "Specialized int: " << value << std::endl;
}
```

Template specialization and instantiation are powerful features that enable highly optimized, type-specific code generation while maintaining the flexibility of generic programming.