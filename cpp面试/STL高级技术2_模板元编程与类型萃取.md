# STL高级技术（二）：模板元编程与类型萃取

## 目录
1. [模板元编程基础](#一模板元编程基础)
2. [SFINAE技术](#二sfinae技术)
3. [类型萃取（Type Traits）](#三类型萃取type-traits)
4. [CRTP模式](#四crtp模式)
5. [策略模式与标签分发](#五策略模式与标签分发)

---

## 一、模板元编程基础

### 1.1 模板元编程的核心概念

模板元编程（Template Metaprogramming, TMP）是利用C++模板系统在编译时执行计算的技术。其核心特征：

1. **编译时计算**：在编译阶段完成计算，零运行时开销
2. **类型作为值**：类型可以作为模板参数传递和操作
3. **递归实例化**：通过模板递归实现循环和条件
4. **代码生成**：根据类型自动生成优化的代码

### 1.2 编译时数值计算

#### 1.2.1 斐波那契数列

```cpp
// 编译时计算斐波那契数列
template<int N>
struct Fibonacci {
    static constexpr int value = Fibonacci<N - 1>::value + Fibonacci<N - 2>::value;
};

// 基础情况
template<>
struct Fibonacci<0> {
    static constexpr int value = 0;
};

template<>
struct Fibonacci<1> {
    static constexpr int value = 1;
};

// 使用
constexpr int fib10 = Fibonacci<10>::value;  // 编译时计算：55
constexpr int fib20 = Fibonacci<20>::value;  // 编译时计算：6765

// C++14变量模板版本
template<int N>
constexpr int fibonacci_v = fibonacci_v<N - 1> + fibonacci_v<N - 2>;

template<>
constexpr int fibonacci_v<0> = 0;

template<>
constexpr int fibonacci_v<1> = 1;
```

#### 1.2.2 阶乘计算

```cpp
// 编译时计算阶乘
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

// 使用
constexpr int fact5 = Factorial<5>::value;  // 120
constexpr int fact10 = Factorial<10>::value; // 3628800

// 使用constexpr函数（C++11）
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// 编译时计算
constexpr int fact7 = factorial(7);  // 5040
```

#### 1.2.3 编译时条件判断

```cpp
// 编译时选择
template<bool Condition, typename TrueType, typename FalseType>
struct Conditional {
    using type = TrueType;
};

template<typename TrueType, typename FalseType>
struct Conditional<false, TrueType, FalseType> {
    using type = FalseType;
};

// 使用
using IntType = Conditional<true, int, double>::type;  // int
using FloatType = Conditional<false, int, double>::type; // double

// C++11的std::conditional
using IntType2 = std::conditional<true, int, double>::type;

// 编译时if（C++17）
template<int N>
auto compile_time_if() {
    if constexpr (N % 2 == 0) {
        return "even";
    } else {
        return "odd";
    }
}
```

### 1.3 类型列表操作

```cpp
// 类型列表基础结构
template<typename... Ts>
struct TypeList {};

// 获取类型列表长度
template<typename List>
struct Length;

template<typename... Ts>
struct Length<TypeList<Ts...>> {
    static constexpr std::size_t value = sizeof...(Ts);
};

// 使用
using MyTypes = TypeList<int, double, char>;
constexpr auto len = Length<MyTypes>::value;  // 3

// 获取第N个类型
template<std::size_t N, typename List>
struct Get;

template<std::size_t N, typename Head, typename... Tail>
struct Get<N, TypeList<Head, Tail...>> {
    using type = typename Get<N - 1, TypeList<Tail...>>::type;
};

template<typename Head, typename... Tail>
struct Get<0, TypeList<Head, Tail...>> {
    using type = Head;
};

// 使用
using FirstType = Get<0, MyTypes>::type;  // int
using SecondType = Get<1, MyTypes>::type; // double

// 添加类型到列表
template<typename List, typename T>
struct PushFront;

template<typename... Ts, typename T>
struct PushFront<TypeList<Ts...>, T> {
    using type = TypeList<T, Ts...>;
};

template<typename List, typename T>
struct PushBack;

template<typename... Ts, typename T>
struct PushBack<TypeList<Ts...>, T> {
    using type = TypeList<Ts..., T>;
};

// 使用
using NewList = PushBack<MyTypes, std::string>::type;
// TypeList<int, double, char, std::string>

// 类型列表遍历
template<typename List, template<typename> class Func>
struct ForEach;

template<typename... Ts, template<typename> class Func>
struct ForEach<TypeList<Ts...>, Func> {
    using type = TypeList<typename Func<Ts>::type...>;
};

// 应用示例：添加const
template<typename T>
struct AddConst {
    using type = const T;
};

using ConstList = ForEach<MyTypes, AddConst>::type;
// TypeList<const int, const double, const char>
```

### 1.4 编译时字符串处理

```cpp
// 编译时字符串长度
template<std::size_t N>
constexpr std::size_t string_length(const char (&)[N]) {
    return N - 1;  // 减去null终止符
}

// 编译时字符串连接
template<std::size_t N1, std::size_t N2>
constexpr auto string_concat(const char (&s1)[N1], const char (&s2)[N2]) {
    std::array<char, N1 + N2 - 1> result{};
    
    for (std::size_t i = 0; i < N1 - 1; ++i) {
        result[i] = s1[i];
    }
    
    for (std::size_t i = 0; i < N2 - 1; ++i) {
        result[N1 - 1 + i] = s2[i];
    }
    
    return result;
}

// 编译时字符串查找
template<std::size_t N>
constexpr std::size_t find_char(const char (&str)[N], char c, std::size_t pos = 0) {
    return pos >= N - 1 ? N - 1 :
           str[pos] == c ? pos :
           find_char(str, c, pos + 1);
}

// 使用
constexpr auto len = string_length("hello");  // 5
constexpr auto concat = string_concat("hello", "world");
constexpr auto pos = find_char("hello", 'l');  // 2
```

### 1.5 编译时算法

```cpp
// 编译时排序（冒泡排序）
template<typename T, std::size_t N>
struct BubbleSort;

template<typename T>
struct BubbleSort<T, 0> {
    static constexpr void sort(T*) {}
};

template<typename T>
struct BubbleSort<T, 1> {
    static constexpr void sort(T* arr) {}
};

template<typename T, std::size_t N>
struct BubbleSort {
    static constexpr void sort(T* arr) {
        bool swapped = false;
        for (std::size_t i = 0; i < N - 1; ++i) {
            if (arr[i] > arr[i + 1]) {
                std::swap(arr[i], arr[i + 1]);
                swapped = true;
            }
        }
        if (swapped) {
            BubbleSort<T, N>::sort(arr);
        }
    }
};

// 编译时二分查找
template<typename T, std::size_t N>
constexpr int binary_search(const T (&arr)[N], const T& target, int left = 0, int right = N - 1) {
    if (left > right) return -1;
    
    int mid = left + (right - left) / 2;
    
    if (arr[mid] == target) return mid;
    else if (arr[mid] > target) return binary_search(arr, target, left, mid - 1);
    else return binary_search(arr, target, mid + 1, right);
}

// 使用
constexpr int arr[] = {3, 1, 4, 1, 5, 9, 2, 6};
constexpr int sorted_arr[sizeof(arr) / sizeof(arr[0])] = {
    1, 1, 2, 3, 4, 5, 6, 9
};
constexpr int pos = binary_search(sorted_arr, 5);  // 4
```

---

## 二、SFINAE技术

### 2.1 SFINAE原理

SFINAE（Substitution Failure Is Not An Error，替换失败并非错误）是C++模板的核心机制之一。当模板参数替换失败时，编译器不会报错，而是丢弃该模板候选。

### 2.2 基础SFINAE应用

#### 2.2.1 类型检测

```cpp
// 检测类型是否有size_type成员
template<typename T>
struct has_size_type {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::declval<typename U::size_type>(), std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

// 检测类型是否有begin()方法
template<typename T>
struct has_begin {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::begin(std::declval<U>()), std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

// 使用
static_assert(has_size_type<std::vector<int>>::value);
static_assert(!has_size_type<int>::value);
static_assert(has_begin<std::vector<int>>::value);
static_assert(!has_begin<int>::value);
```

#### 2.2.2 enable_if

```cpp
// 基础enable_if实现
template<bool B, typename T = void>
struct enable_if {};

template<typename T>
struct enable_if<true, T> {
    using type = T;
};

// 使用enable_if约束函数
template<typename T>
typename enable_if<std::is_integral<T>::value, T>::type
add(T a, T b) {
    return a + b;
}

template<typename T>
typename enable_if<std::is_floating_point<T>::value, T>::type
add(T a, T b) {
    return a + b;
}

// 使用
int result1 = add(10, 20);      // 调用整数版本
double result2 = add(1.5, 2.5);  // 调用浮点版本
// add("hello", "world");        // 编译错误：无匹配函数

// C++14的enable_if_t
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
multiply(T a, T b) {
    return a * b;
}
```

### 2.3 高级SFINAE技术

#### 2.3.1 检测成员函数存在性

```cpp
// 检测类型是否有reserve方法
template<typename T>
struct has_reserve {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::declval<U>().reserve(0), std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

// 根据容器是否有reserve优化插入
template<typename Container>
void reserve_if_possible(Container& c, std::size_t n) {
    if constexpr (has_reserve<Container>::value) {
        c.reserve(n);
    }
}

// 使用
std::vector<int> vec;
reserve_if_possible(vec, 100);  // 会调用reserve

std::list<int> lst;
reserve_if_possible(lst, 100);  // 不会调用reserve
```

#### 2.3.2 检测表达式有效性

```cpp
// 检测是否可以相加
template<typename T, typename U>
struct is_addable {
private:
    template<typename A, typename B>
    static auto test(int) -> decltype(
        std::declval<A>() + std::declval<B>(), std::true_type{}
    );
    
    template<typename, typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T, U>(0))::value;
};

// 检测是否可以比较
template<typename T, typename U>
struct is_comparable {
private:
    template<typename A, typename B>
    static auto test(int) -> decltype(
        std::declval<A>() < std::declval<B>(), std::true_type{}
    );
    
    template<typename, typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T, U>(0))::value;
};

// 使用
static_assert(is_addable<int, int>::value);
static_assert(!is_addable<std::string, int>::value);
static_assert(is_comparable<int, int>::value);
static_assert(!is_comparable<std::string, std::vector<int>>::value);
```

### 2.4 SFINAE面试问题

#### 面试问题6：SFINAE的作用是什么？如何避免滥用？

**答案：**
SFINAE的作用：

1. **重载决议**：根据类型特性选择正确的函数
2. **类型约束**：编译时检查类型是否满足要求
3. **条件编译**：根据类型特性生成不同代码

避免滥用：

1. **使用Concepts（C++20）**：更清晰、更易读
2. **避免过度嵌套**：使代码难以理解
3. **提供清晰的错误信息**：避免晦涩的编译错误

```cpp
// SFINAE方式（复杂）
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process(T value) {
    std::cout << "Integral: " << value << std::endl;
}

template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process(T value) {
    std::cout << "Floating: " << value << std::endl;
}

// Concepts方式（C++20，更清晰）
template<typename T>
concept Integral = std::is_integral_v<T>;

template<typename T>
concept FloatingPoint = std::is_floating_point_v<T>;

template<Integral T>
void process(T value) {
    std::cout << "Integral: " << value << std::endl;
}

template<FloatingPoint T>
void process(T value) {
    std::cout << "Floating: " << value << std::endl;
}
```

---

## 三、类型萃取（Type Traits）

### 3.1 类型萃取基础

类型萃取是编译时获取类型信息的机制，是模板元编程的基础工具。

### 3.2 基础类型萃取实现

#### 3.2.1 主要类型分类

```cpp
// 移除引用
template<typename T>
struct remove_reference {
    using type = T;
};

template<typename T>
struct remove_reference<T&> {
    using type = T;
};

template<typename T>
struct remove_reference<T&&> {
    using type = T;
};

// 移除const
template<typename T>
struct remove_const {
    using type = T;
};

template<typename T>
struct remove_const<const T> {
    using type = T;
};

// 移除指针
template<typename T>
struct remove_pointer {
    using type = T;
};

template<typename T>
struct remove_pointer<T*> {
    using type = T;
};

template<typename T>
struct remove_pointer<T* const> {
    using type = T;
};

// 添加指针
template<typename T>
struct add_pointer {
    using type = typename remove_reference<T>::type*;
};

// 添加引用
template<typename T>
struct add_lvalue_reference {
    using type = T&;
};

template<typename T>
struct add_rvalue_reference {
    using type = T&&;
};

// 使用
using T1 = remove_reference<int&>::type;        // int
using T2 = remove_const<const int>::type;        // int
using T3 = remove_pointer<int*>::type;           // int
using T4 = add_pointer<int>::type;               // int*
using T5 = add_lvalue_reference<int>::type;      // int&
```

#### 3.2.2 类型特性判断

```cpp
// 是否为const
template<typename T>
struct is_const : std::false_type {};

template<typename T>
struct is_const<const T> : std::true_type {};

// 是否为引用
template<typename T>
struct is_reference : std::false_type {};

template<typename T>
struct is_reference<T&> : std::true_type {};

template<typename T>
struct is_reference<T&&> : std::true_type {};

// 是否为指针
template<typename T>
struct is_pointer : std::false_type {};

template<typename T>
struct is_pointer<T*> : std::true_type {};

template<typename T>
struct is_pointer<T* const> : std::true_type {};

// 是否为数组
template<typename T>
struct is_array : std::false_type {};

template<typename T, std::size_t N>
struct is_array<T[N]> : std::true_type {};

template<typename T>
struct is_array<T[]> : std::true_type {};

// 是否为函数
template<typename T>
struct is_function : std::false_type {};

template<typename Ret, typename... Args>
struct is_function<Ret(Args...)> : std::true_type {};

template<typename Ret, typename... Args>
struct is_function<Ret(Args..., ...)> : std::true_type {};  // C风格可变参数

// 使用
static_assert(is_const<const int>::value);
static_assert(!is_const<int>::value);
static_assert(is_reference<int&>::value);
static_assert(is_pointer<int*>::value);
static_assert(is_array<int[10]>::value);
static_assert(is_function<void(int, double)>::value);
```

### 3.3 类型转换萃取

```cpp
// decay类型（数组转指针、函数转函数指针、移除cv和引用）
template<typename T>
struct decay {
private:
    using U = typename remove_reference<T>::type;
public:
    using type = typename std::conditional<
        is_array<U>::value,
        typename remove_extent<U>::type*,
        typename std::conditional<
            is_function<U>::value,
            typename add_pointer<U>::type,
            typename remove_cv<U>::type
        >::type
    >::type;
};

// common_type：找到所有类型的公共类型
template<typename... Ts>
struct common_type;

template<typename T>
struct common_type<T> {
    using type = T;
};

template<typename T1, typename T2>
struct common_type<T1, T2> {
    using type = decltype(true ? std::declval<T1>() : std::declval<T2>());
};

template<typename T1, typename T2, typename... Ts>
struct common_type<T1, T2, Ts...> {
    using type = typename common_type<
        typename common_type<T1, T2>::type,
        Ts...
    >::type;
};

// 使用
using DecayType1 = decay<int(&)[10]>::type;      // int*
using DecayType2 = decay<void(int)>::type;        // void(*)(int)
using DecayType3 = decay<const int&>::type;       // int

using CommonType = common_type<int, double, float>::type;  // double
```

### 3.4 类型关系萃取

```cpp
// is_same：判断两个类型是否相同
template<typename T, typename U>
struct is_same : std::false_type {};

template<typename T>
struct is_same<T, T> : std::true_type {};

// is_base_of：判断是否为基类
template<typename Base, typename Derived>
struct is_base_of : std::is_base_of<Base, Derived> {};

// is_convertible：判断是否可以转换
template<typename From, typename To>
struct is_convertible : std::is_convertible<From, To> {};

// is_constructible：判断是否可构造
template<typename T, typename... Args>
struct is_constructible : std::is_constructible<T, Args...> {};

// is_assignable：判断是否可赋值
template<typename T, typename U>
struct is_assignable : std::is_assignable<T, U> {};

// 使用
static_assert(is_same<int, int>::value);
static_assert(!is_same<int, double>::value);
static_assert(is_base_of<std::ios, std::ostream>::value);
static_assert(is_convertible<int, double>::value);
static_assert(is_constructible<std::string, const char*>::value);
```

### 3.5 类型操作萃取

```cpp
// 条件类型选择
template<bool B, typename T, typename F>
struct conditional {
    using type = T;
};

template<typename T, typename F>
struct conditional<false, T, F> {
    using type = F;
};

// 启用/禁用类型
template<bool B, typename T = void>
struct enable_if {};

template<typename T>
struct enable_if<true, T> {
    using type = T;
};

// 类型选择
template<typename T, typename U>
struct type_identity {
    using type = T;
};

// 使用
using CondType = conditional<true, int, double>::type;  // int
using EnabledType = enable_if<true, int>::type;         // int
// using DisabledType = enable_if<false, int>::type;    // 无type成员
```

### 3.6 高级类型萃取应用

```cpp
// 完美转发辅助
template<typename T>
struct decay_copy {
    using type = typename std::decay<T>::type;
};

template<typename T>
using decay_copy_t = typename decay_copy<T>::type;

// 检测是否有迭代器
template<typename T>
struct has_iterator {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::begin(std::declval<U>()) != std::end(std::declval<U>()),
        ++std::declval<decltype(std::begin(std::declval<U>()))>(),
        *std::begin(std::declval<U>()),
        std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

// 检测是否为容器
template<typename T>
struct is_container {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::declval<U>().size(),
        std::declval<U>().begin(),
        std::declval<U>().end(),
        std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

// 使用
static_assert(has_iterator<std::vector<int>>::value);
static_assert(!has_iterator<int>::value);
static_assert(is_container<std::vector<int>>::value);
static_assert(is_container<std::string>::value);
```

---

## 四、CRTP模式

### 4.1 CRTP原理

CRTP（Curiously Recurring Template Pattern，奇异递归模板模式）是一种C++惯用法，派生类将自己作为模板参数传递给基类。

### 4.2 CRTP基础应用

#### 4.2.1 静态多态

```cpp
// CRTP基类
template<typename Derived>
class Shape {
public:
    void draw() const {
        static_cast<const Derived*>(this)->draw_impl();
    }
    
    double area() const {
        return static_cast<const Derived*>(this)->area_impl();
    }
};

// 派生类
class Circle : public Shape<Circle> {
public:
    Circle(double radius) : radius_(radius) {}
    
    void draw_impl() const {
        std::cout << "Drawing circle with radius " << radius_ << std::endl;
    }
    
    double area_impl() const {
        return 3.14159 * radius_ * radius_;
    }
    
private:
    double radius_;
};

class Rectangle : public Shape<Rectangle> {
public:
    Rectangle(double width, double height) 
        : width_(width), height_(height) {}
    
    void draw_impl() const {
        std::cout << "Drawing rectangle " << width_ << "x" << height_ << std::endl;
    }
    
    double area_impl() const {
        return width_ * height_;
    }
    
private:
    double width_, height_;
};

// 使用
void process_shape(const Shape<auto>& shape) {
    shape.draw();
    std::cout << "Area: " << shape.area() << std::endl;
}

Circle circle(5.0);
Rectangle rectangle(3.0, 4.0);

process_shape(circle);     // 静态多态调用
process_shape(rectangle); // 静态多态调用
```

#### 4.2.2 接口混入

```cpp
// 单例模式混入
template<typename Derived>
class Singleton {
public:
    static Derived& instance() {
        static Derived inst;
        return inst;
    }
    
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    
protected:
    Singleton() = default;
    ~Singleton() = default;
};

// 可复制对象混入
template<typename Derived>
class Cloneable {
public:
    std::unique_ptr<Derived> clone() const {
        return std::make_unique<Derived>(
            static_cast<const Derived&>(*this)
        );
    }
    
    std::unique_ptr<Derived> clone() {
        return std::make_unique<Derived>(
            static_cast<Derived&>(*this)
        );
    }
};

// 可比较对象混入
template<typename Derived>
class Comparable {
public:
    bool operator==(const Comparable& other) const {
        return static_cast<const Derived*>(this)->compare(
            *static_cast<const Derived*>(&other)
        ) == 0;
    }
    
    bool operator!=(const Comparable& other) const {
        return !(*this == other);
    }
    
    bool operator<(const Comparable& other) const {
        return static_cast<const Derived*>(this)->compare(
            *static_cast<const Derived*>(&other)
        ) < 0;
    }
};

// 使用混入
class MyClass : public Singleton<MyClass>,
                public Cloneable<MyClass>,
                public Comparable<MyClass> {
public:
    MyClass(int value) : value_(value) {}
    
    int compare(const MyClass& other) const {
        if (value_ < other.value_) return -1;
        if (value_ > other.value_) return 1;
        return 0;
    }
    
    void print() const {
        std::cout << "Value: " << value_ << std::endl;
    }
    
private:
    int value_;
};

// 使用
MyClass& obj = MyClass::instance();
auto clone = obj.clone();

MyClass obj1(10);
MyClass obj2(20);
bool equal = (obj1 == obj2);  // false
bool less = (obj1 < obj2);    // true
```

### 4.3 CRTP高级应用

#### 4.3.1 运算符自动生成

```cpp
// 自动生成比较运算符
template<typename Derived>
class ComparisonOperators {
public:
    bool operator!=(const ComparisonOperators& other) const {
        return !(*static_cast<const Derived*>(this) == 
                static_cast<const Derived&>(other));
    }
    
    bool operator>(const ComparisonOperators& other) const {
        return other < *static_cast<const Derived*>(this);
    }
    
    bool operator<=(const ComparisonOperators& other) const {
        return !(*static_cast<const Derived*>(this) > 
                 static_cast<const Derived&>(other));
    }
    
    bool operator>=(const ComparisonOperators& other) const {
        return !(*static_cast<const Derived*>(this) < 
                 static_cast<const Derived&>(other));
    }
};

// 使用
class Point : public ComparisonOperators<Point> {
public:
    Point(int x, int y) : x_(x), y_(y) {}
    
    bool operator<(const Point& other) const {
        if (x_ != other.x_) return x_ < other.x_;
        return y_ < other.y_;
    }
    
    bool operator==(const Point& other) const {
        return x_ == other.x_ && y_ == other.y_;
    }
    
private:
    int x_, y_;
};

Point p1(1, 2);
Point p2(3, 4);

bool lt = (p1 < p2);    // true
bool gt = (p1 > p2);    // false
bool ne = (p1 != p2);   // true
bool le = (p1 <= p2);   // true
bool ge = (p1 >= p2);   // false
```

#### 4.3.2 接口注册

```cpp
// 类型注册系统
template<typename Derived>
class InterfaceRegistry {
private:
    static std::unordered_map<std::string, std::function<Derived()>> creators_;
    
public:
    static bool register_type(const std::string& name, 
                              std::function<Derived()> creator) {
        creators_[name] = creator;
        return true;
    }
    
    static std::unique_ptr<Derived> create(const std::string& name) {
        auto it = creators_.find(name);
        if (it != creators_.end()) {
            return std::make_unique<Derived>(it->second());
        }
        return nullptr;
    }
    
    static std::vector<std::string> registered_types() {
        std::vector<std::string> types;
        for (const auto& pair : creators_) {
            types.push_back(pair.first);
        }
        return types;
    }
};

template<typename Derived>
std::unordered_map<std::string, std::function<Derived()>> 
InterfaceRegistry<Derived>::creators_;

// 使用
class Shape : public InterfaceRegistry<Shape> {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
};

class Circle : public Shape {
public:
    void draw() const override {
        std::cout << "Circle" << std::endl;
    }
};

class Rectangle : public Shape {
public:
    void draw() const override {
        std::cout << "Rectangle" << std::endl;
    }
};

// 注册类型
namespace {
    bool circle_registered = Shape::register_type("Circle", []() {
        return Circle();
    });
    
    bool rectangle_registered = Shape::register_type("Rectangle", []() {
        return Rectangle();
    });
}

// 创建对象
auto circle = Shape::create("Circle");
auto rectangle = Shape::create("Rectangle");

if (circle) circle->draw();
if (rectangle) rectangle->draw();
```

### 4.4 CRTP面试问题

#### 面试问题7：CRTP相比虚函数有什么优势？

**答案：**
CRTP的优势：

1. **零运行时开销**：静态分发，无虚函数表查找
2. **内联优化**：编译器可以内联所有调用
3. **编译时多态**：类型安全，编译时检查
4. **代码生成**：可以为每个类型生成特化代码

虚函数的优势：

1. **运行时多态**：真正的运行时类型绑定
2. **动态类型**：可以处理运行时确定的类型
3. **简单直观**：语法更简单

```cpp
// CRTP：静态多态
template<typename Derived>
class StaticPolymorphism {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

class CRTPDerived : public StaticPolymorphism<CRTPDerived> {
public:
    void implementation() {
        std::cout << "CRTP implementation" << std::endl;
    }
};

// 虚函数：动态多态
class DynamicPolymorphism {
public:
    virtual void interface() = 0;
    virtual ~DynamicPolymorphism() = default;
};

class VirtualDerived : public DynamicPolymorphism {
public:
    void interface() override {
        std::cout << "Virtual implementation" << std::endl;
    }
};

// 性能比较
// CRTP：直接调用，可内联
CRTPDerived crtp_obj;
crtp_obj.interface();  // 编译时确定，零开销

// 虚函数：间接调用，通过vtable
DynamicPolymorphism* virtual_obj = new VirtualDerived();
virtual_obj->interface();  // 运行时查找，有开销
delete virtual_obj;
```

---

## 五、策略模式与标签分发

### 5.1 标签分发技术

标签分发（Tag Dispatching）是一种基于类型标签选择不同实现的编译时多态技术。

#### 5.1.1 基础标签分发

```cpp
// 定义标签
struct random_access_iterator_tag {};
struct bidirectional_iterator_tag {};
struct forward_iterator_tag {};
struct input_iterator_tag {};
struct output_iterator_tag {};

// 迭代器萃取
template<typename Iterator>
struct iterator_traits {
    using iterator_category = typename Iterator::iterator_category;
    using value_type = typename Iterator::value_type;
    using difference_type = typename Iterator::difference_type;
    using pointer = typename Iterator::pointer;
    using reference = typename Iterator::reference;
};

// 根据迭代器类型选择不同算法
template<typename Iterator, typename Distance>
void advance_impl(Iterator& it, Distance n, random_access_iterator_tag) {
    it += n;  // O(1)
}

template<typename Iterator, typename Distance>
void advance_impl(Iterator& it, Distance n, bidirectional_iterator_tag) {
    if (n > 0) {
        for (; n > 0; --n, ++it) {}
    } else {
        for (; n < 0; ++n, --it) {}
    }
    // O(n)
}

template<typename Iterator, typename Distance>
void advance_impl(Iterator& it, Distance n, input_iterator_tag) {
    for (; n > 0; --n, ++it) {}
    // O(n)
}

// 标签分发入口
template<typename Iterator, typename Distance>
void advance(Iterator& it, Distance n) {
    advance_impl(it, n, 
                typename iterator_traits<Iterator>::iterator_category{});
}
```

#### 5.1.2 自定义标签分发

```cpp
// 快速排序标签
struct quick_sort_tag {};
struct merge_sort_tag {};
struct insertion_sort_tag {};

// 根据容器大小选择排序算法
template<typename Iterator>
void sort_impl(Iterator first, Iterator last, quick_sort_tag) {
    std::cout << "Using quick sort" << std::endl;
    // 快速排序实现
}

template<typename Iterator>
void sort_impl(Iterator first, Iterator last, merge_sort_tag) {
    std::cout << "Using merge sort" << std::endl;
    // 归并排序实现
}

template<typename Iterator>
void sort_impl(Iterator first, Iterator last, insertion_sort_tag) {
    std::cout << "Using insertion sort" << std::endl;
    // 插入排序实现
}

// 选择排序算法
template<typename Iterator>
void smart_sort(Iterator first, Iterator last) {
    auto size = std::distance(first, last);
    
    if (size < 10) {
        sort_impl(first, last, insertion_sort_tag{});
    } else if (size < 1000) {
        sort_impl(first, last, quick_sort_tag{});
    } else {
        sort_impl(first, last, merge_sort_tag{});
    }
}

// 使用
std::vector<int> vec = {5, 2, 8, 1, 9};
smart_sort(vec.begin(), vec.end());
```

### 5.2 策略模式

策略模式允许在运行时或编译时选择算法。

#### 5.2.1 编译时策略

```cpp
// 排序策略
template<typename Strategy>
class Sorter {
public:
    template<typename Iterator>
    void sort(Iterator first, Iterator last) {
        Strategy::sort(first, last);
    }
};

// 快速排序策略
struct QuickSortStrategy {
    template<typename Iterator>
    static void sort(Iterator first, Iterator last) {
        std::cout << "QuickSort" << std::endl;
        std::sort(first, last);
    }
};

// 归并排序策略
struct MergeSortStrategy {
    template<typename Iterator>
    static void sort(Iterator first, Iterator last) {
        std::cout << "MergeSort" << std::endl;
        std::stable_sort(first, last);
    }
};

// 使用
Sorter<QuickSortStrategy> quick_sorter;
Sorter<MergeSortStrategy> merge_sorter;

std::vector<int> vec = {3, 1, 4, 1, 5};
quick_sorter.sort(vec.begin(), vec.end());
merge_sorter.sort(vec.begin(), vec.end());
```

#### 5.2.2 运行时策略

```cpp
// 运行时策略接口
class SortStrategy {
public:
    virtual ~SortStrategy() = default;
    virtual void sort(std::vector<int>& vec) = 0;
};

// 具体策略
class QuickSort : public SortStrategy {
public:
    void sort(std::vector<int>& vec) override {
        std::cout << "QuickSort" << std::endl;
        std::sort(vec.begin(), vec.end());
    }
};

class MergeSort : public SortStrategy {
public:
    void sort(std::vector<int>& vec) override {
        std::cout << "MergeSort" << std::endl;
        std::stable_sort(vec.begin(), vec.end());
    }
};

// 策略使用者
class Sorter {
public:
    Sorter(std::unique_ptr<SortStrategy> strategy) 
        : strategy_(std::move(strategy)) {}
    
    void sort(std::vector<int>& vec) {
        strategy_->sort(vec);
    }
    
    void set_strategy(std::unique_ptr<SortStrategy> strategy) {
        strategy_ = std::move(strategy);
    }
    
private:
    std::unique_ptr<SortStrategy> strategy_;
};

// 使用
std::vector<int> vec = {3, 1, 4, 1, 5};
Sorter sorter(std::make_unique<QuickSort>());
sorter.sort(vec);

sorter.set_strategy(std::make_unique<MergeSort>());
sorter.sort(vec);
```

### 5.3 混合技术应用

```cpp
// 结合CRTP、策略模式和标签分发
template<typename Derived, typename SortStrategy>
class DataProcessor : public SortStrategy {
public:
    void process() {
        auto& derived = static_cast<Derived&>(*this);
        auto data = derived.get_data();
        
        // 使用标签分发选择最优算法
        sort_data(data.begin(), data.end(), 
                 typename get_sort_tag<decltype(data)>::type{});
        
        derived.process_result(data);
    }
    
private:
    template<typename Iterator, typename Tag>
    void sort_data(Iterator first, Iterator last, Tag tag) {
        SortStrategy::sort(first, last);
    }
};

// 数据类型萃取
template<typename Container>
struct get_sort_tag {
    using type = quick_sort_tag;
};

template<typename T>
struct get_sort_tag<std::list<T>> {
    using type = merge_sort_tag;
};

// 具体处理器
class IntegerProcessor : public DataProcessor<IntegerProcessor, QuickSortStrategy> {
public:
    std::vector<int> get_data() {
        return {5, 2, 8, 1, 9};
    }
    
    void process_result(const std::vector<int>& data) {
        std::cout << "Processed: ";
        for (int val : data) {
            std::cout << val << " ";
        }
        std::cout << std::endl;
    }
};

// 使用
IntegerProcessor processor;
processor.process();
```

---

## 总结

本文档深入讲解了STL中的模板元编程与类型萃取技术，包括：

1. **模板元编程基础**：编译时计算、类型列表操作、编译时算法
2. **SFINAE技术**：类型检测、enable_if、表达式有效性检测
3. **类型萃取**：基础类型萃取、类型转换、类型关系、高级应用
4. **CRTP模式**：静态多态、接口混入、运算符自动生成
5. **策略模式与标签分发**：编译时策略、运行时策略、混合技术

这些技术是现代C++编程的核心，掌握它们对深入理解STL和编写高性能代码至关重要。