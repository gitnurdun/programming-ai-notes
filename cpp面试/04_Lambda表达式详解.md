# Lambda表达式详解 (C++11-20)

## 一、Lambda表达式概述

### 1.1 什么是Lambda表达式

Lambda表达式是C++11引入的一种匿名函数语法，允许在代码中直接定义和使用函数对象。它提供了一种简洁、灵活的方式来创建可调用对象。

**核心特点：**
- 匿名函数：无需显式命名
- 闭包：可以捕获外部变量
- 内联：可以内联优化
- 类型推导：自动推导返回类型

### 1.2 Lambda表达式的历史演进

| 版本 | 新特性 |
|------|--------|
| C++11 | 基础Lambda语法 |
| C++14 | 泛型Lambda、初始化捕获 |
| C++17 | constexpr Lambda |
| C++20 | 模板Lambda、consteval Lambda |

## 二、Lambda表达式基础语法

### 2.1 基本语法

```cpp
// 基本形式
[capture](parameters) -> return_type { body }

// 示例
auto add = [](int a, int b) -> int {
    return a + b;
};

// 简化形式（自动推导返回类型）
auto multiply = [](int a, int b) {
    return a * b;
};

// 使用
int result = add(3, 4);  // 7
```

### 2.2 语法组成

```cpp
[捕获列表](参数列表) mutable 异常说明 -> 返回类型 { 函数体 }

// 详细示例
auto lambda = 
    [capture]                      // 捕获列表
    (int x, int y)                 // 参数列表
    mutable                        // mutable说明符
    noexcept                       // 异常说明
    -> double                      // 返回类型
    {                              // 函数体
        return static_cast<double>(x) / y;
    };
```

## 三、捕获列表（Capture List）

### 3.1 捕获方式

```cpp
// 1. 值捕获（按值）
int x = 10;
auto lambda1 = [x]() { return x * 2; };
x = 20;
std::cout << lambda1();  // 输出20（捕获时的值）

// 2. 引用捕获（按引用）
int y = 10;
auto lambda2 = [&y]() { return y * 2; };
y = 20;
std::cout << lambda2();  // 输出40（当前值）

// 3. 隐式捕获
int a = 1, b = 2, c = 3;
auto lambda3 = [=]() { return a + b + c; };      // 按值捕获所有
auto lambda4 = [&]() { return a + b + c; };      // 按引用捕获所有

// 4. 混合捕获
int m = 10, n = 20;
auto lambda5 = [m, &n]() { return m + n; };      // m按值，n按引用
auto lambda6 = [=, &n]() { return m + n; };      // m按值，n按引用
auto lambda7 = [&, m]() { return m + n; };       // n按引用，m按值
```

### 3.2 捕获示例

```cpp
// 捕获this指针
class MyClass {
public:
    void method() {
        auto lambda = [this]() {
            return this->value_;
        };
    }
    
private:
    int value_ = 42;
};

// C++17: 捕获*this（按值）
auto lambda8 = [*this]() {
    return this->value_;  // this指向拷贝的对象
};

// C++14: 初始化捕获
int value = 42;
auto lambda9 = [x = value + 10]() {
    return x;  // x = 52
};

// 移动捕获
std::string str = "hello";
auto lambda10 = [s = std::move(str)]() {
    return s;  // s是移动构造的
};
```

### 3.3 捕获注意事项

```cpp
// 注意：捕获的值在Lambda创建时确定
int count = 0;
auto lambda = [count]() mutable {
    ++count;  // 修改的是拷贝
    return count;
};

std::cout << lambda();  // 1
std::cout << lambda();  // 2
std::cout << count;     // 0（外部count未改变）

// 引用捕获可以修改外部变量
int total = 0;
auto accumulator = [&total](int x) {
    total += x;
};

accumulator(10);
accumulator(20);
std::cout << total;  // 30
```

## 四、Lambda表达式参数

### 4.1 参数列表

```cpp
// 无参数
auto lambda1 = []() { return 42; };

// 单个参数
auto lambda2 = [](int x) { return x * 2; };

// 多个参数
auto lambda3 = [](int a, int b, int c) {
    return a + b + c;
};

// 使用auto（泛型Lambda）
auto lambda4 = [](auto x, auto y) {
    return x + y;
};

// 默认参数（C++14）
auto lambda5 = [](int x, int y = 10) {
    return x + y;
};

// 可变参数（C++11）
auto lambda6 = [](auto... args) {
    return (args + ...);  // C++17折叠表达式
};
```

### 4.2 参数传递

```cpp
// 按值传递
auto by_value = [](std::string s) {
    return s;
};

// 按引用传递
auto by_ref = [](std::string& s) {
    s += " modified";
};

// 按const引用传递
auto by_const_ref = [](const std::string& s) {
    return s.length();
};

// 按右值引用传递
auto by_rvalue_ref = [](std::string&& s) {
    return std::move(s);
};

// 使用完美转发
auto perfect_forward = [](auto&& x) {
    return process(std::forward<decltype(x)>(x));
};
```

## 五、Lambda表达式返回类型

### 5.1 返回类型推导

```cpp
// 自动推导
auto lambda1 = [](int x) {
    return x * 2;  // 推导为int
};

// 显式指定返回类型
auto lambda2 = [](int x) -> double {
    return x * 2;  // 返回double
};

// 尾返回类型（C++11）
auto lambda3 = [](int x) -> decltype(x * 2) {
    return x * 2;
};

// 需要显式指定返回类型的情况
auto lambda4 = [](bool condition) -> std::string {
    if (condition) {
        return "yes";  // 返回std::string
    }
    return "no";       // 返回std::string
};
```

### 5.2 返回类型复杂示例

```cpp
// 返回复杂类型
auto lambda5 = []() -> std::vector<int> {
    return {1, 2, 3, 4, 5};
};

// 返回Lambda（高阶函数）
auto make_multiplier = [](int factor) {
    return [factor](int x) {
        return x * factor;
    };
};

auto times3 = make_multiplier(3);
std::cout << times3(5);  // 15

// 返回std::function
std::function<int(int)> create_function() {
    return [](int x) { return x * x; };
}
```

## 六、Lambda表达式的高级特性

### 6.1 mutable关键字

```cpp
// mutable允许修改按值捕获的变量
int x = 10;
auto lambda = [x]() mutable {
    x = 20;  // OK：修改的是拷贝
    return x;
};

std::cout << lambda();  // 20
std::cout << x;         // 10（外部x未改变）

// 使用mutable实现状态
auto counter = [count = 0]() mutable {
    return ++count;
};

std::cout << counter();  // 1
std::cout << counter();  // 2
```

### 6.2 异常说明

```cpp
// noexcept
auto lambda1 = []() noexcept {
    return 42;
};

// 条件noexcept
auto lambda2 = []() noexcept(noexcept(some_function())) {
    return some_function();
};

// 指定异常类型
auto lambda3 = []() throw(std::runtime_error) {
    throw std::runtime_error("Error");
};
```

### 6.3 constexpr Lambda (C++17)

```cpp
// constexpr Lambda可以在编译时求值
constexpr auto square = [](int x) {
    return x * x;
};

// 编译时计算
constexpr int result = square(5);  // 25
static_assert(square(5) == 25);

// 在模板中使用
template<int N>
constexpr int factorial = N <= 1 ? 1 : N * factorial<N - 1>;

constexpr auto factorial_lambda = [](int n) constexpr {
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
};

static_assert(factorial_lambda(5) == 120);
```

### 6.4 consteval Lambda (C++20)

```cpp
// consteval Lambda必须在编译时求值
consteval auto compile_time_add(int a, int b) {
    return a + b;
}

// 编译时计算
constexpr int sum = compile_time_add(10, 20);  // OK
// int runtime_sum = compile_time_add(10, 20); // 错误：不是常量表达式
```

## 七、泛型Lambda (C++14)

### 7.1 基本泛型Lambda

```cpp
// 使用auto参数
auto generic_add = [](auto a, auto b) {
    return a + b;
};

// 可以接受不同类型
std::cout << generic_add(1, 2);        // 3 (int)
std::cout << generic_add(1.5, 2.5);    // 4.0 (double)
std::cout << generic_add(std::string("Hello "), "World");  // "Hello World"

// 泛型Lambda返回类型
auto generic_cast = [](auto value) -> decltype(auto) {
    return std::move(value);
};
```

### 7.2 泛型Lambda高级用法

```cpp
// 多个auto参数
auto compare = [](auto a, auto b) {
    if constexpr (std::is_integral_v<decltype(a)>) {
        return a <=> b;  // 使用三相比较
    } else {
        return a < b;
    }
};

// 使用if constexpr
auto process = [](auto value) {
    if constexpr (std::is_integral_v<decltype(value)>) {
        std::cout << "Integral: " << value << std::endl;
    } else if constexpr (std::is_floating_point_v<decltype(value)>) {
        std::cout << "Floating: " << value << std::endl;
    } else {
        std::cout << "Other: " << value << std::endl;
    }
};

// 完美转发
auto forwarder = [](auto&&... args) {
    return some_function(std::forward<decltype(args)>(args)...);
};
```

## 八、模板Lambda (C++20)

### 8.1 模板Lambda语法

```cpp
// 模板Lambda
auto lambda = []<typename T>(T x) {
    return x * 2;
};

// 多个模板参数
auto lambda2 = []<typename T, typename U>(T x, U y) {
    return x + y;
};

// 模板参数约束
auto lambda3 = []<std::integral T>(T x) {
    return x * 2;
};

// 使用
std::cout << lambda(5);       // 10
std::cout << lambda2(1, 2);   // 3
std::cout << lambda2(1.5, 2.5); // 4.0
```

### 8.2 模板Lambda高级用法

```cpp
// 使用requires约束
auto constrained_lambda = []<typename T>(T x) 
    requires std::integral<T>
{
    return x * 2;
};

// 使用auto和模板参数混合
auto mixed_lambda = []<typename T>(T x, auto y) {
    return x + y;
};

// 变参模板Lambda
auto variadic_lambda = []<typename... Ts>(Ts... args) {
    return (args + ...);
};

// 使用
std::cout << variadic_lambda(1, 2, 3, 4, 5);  // 15
```

## 九、Lambda表达式的应用

### 9.1 算法库应用

```cpp
#include <algorithm>
#include <vector>

std::vector<int> numbers = {1, 5, 3, 8, 2, 9};

// 排序
std::sort(numbers.begin(), numbers.end(), [](int a, int b) {
    return a > b;  // 降序排序
});

// 查找
auto it = std::find_if(numbers.begin(), numbers.end(), [](int x) {
    return x > 5;
});

// 转换
std::vector<int> doubled;
std::transform(numbers.begin(), numbers.end(), std::back_inserter(doubled),
    [](int x) { return x * 2; });

// 统计
int count = std::count_if(numbers.begin(), numbers.end(), [](int x) {
    return x % 2 == 0;
});

// 删除
auto new_end = std::remove_if(numbers.begin(), numbers.end(), [](int x) {
    return x < 5;
});
numbers.erase(new_end, numbers.end());
```

### 9.2 回调函数

```cpp
// 异步操作
void async_operation(std::function<void(int)> callback) {
    // 模拟异步操作
    std::thread([callback]() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        callback(42);
    }).detach();
}

// 使用Lambda作为回调
async_operation([](int result) {
    std::cout << "Result: " << result << std::endl;
});

// 捕获上下文
int counter = 0;
async_operation([counter](int result) mutable {
    counter += result;
    std::cout << "Counter: " << counter << std::endl;
});
```

### 9.3 事件处理

```cpp
// 事件系统
class Event {
public:
    using Handler = std::function<void(int)>;
    
    void subscribe(Handler handler) {
        handlers_.push_back(handler);
    }
    
    void fire(int value) {
        for (auto& handler : handlers_) {
            handler(value);
        }
    }
    
private:
    std::vector<Handler> handlers_;
};

// 使用
Event event;
event.subscribe([](int value) {
    std::cout << "Handler 1: " << value << std::endl;
});

event.subscribe([](int value) {
    std::cout << "Handler 2: " << value * 2 << std::endl;
});

event.fire(10);
```

### 9.4 函数组合

```cpp
// 函数组合
template<typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto x) {
        return f(g(x));
    };
}

// 使用
auto square = [](int x) { return x * x; };
auto increment = [](int x) { return x + 1; };

auto square_then_increment = compose(increment, square);
auto increment_then_square = compose(square, increment);

std::cout << square_then_increment(5);    // 26 (5*5 + 1)
std::cout << increment_then_square(5);    // 36 ((5+1)*(5+1))
```

### 9.5 状态管理

```cpp
// 使用Lambda创建状态机
class StateMachine {
public:
    StateMachine() {
        state_ = [](auto& sm, int event) {
            std::cout << "Initial state, event: " << event << std::endl;
            sm.state_ = &StateMachine::running_state;
        };
    }
    
    void handle(int event) {
        state_(*this, event);
    }
    
private:
    std::function<void(StateMachine&, int)> state_;
    
    static void running_state(StateMachine& sm, int event) {
        std::cout << "Running state, event: " << event << std::endl;
        if (event == 0) {
            sm.state_ = &StateMachine::initial_state;
        }
    }
    
    static void initial_state(StateMachine& sm, int event) {
        std::cout << "Initial state, event: " << event << std::endl;
        sm.state_ = &StateMachine::running_state;
    }
};
```

## 十、Lambda表达式的底层原理

### 10.1 闭包类型

```cpp
// Lambda表达式生成一个唯一的闭包类型
auto lambda = [](int x) { return x * 2; };

// 等价于：
class LambdaClosureType {
public:
    int operator()(int x) const {
        return x * 2;
    }
};

// LambdaClosureType lambda;
```

### 10.2 捕获的实现

```cpp
// 值捕获
int x = 10;
auto lambda = [x]() { return x; };

// 等价于：
class LambdaClosureType {
public:
    LambdaClosureType(int captured_x) : x_(captured_x) {}
    
    int operator()() const {
        return x_;
    }
    
private:
    int x_;
};

LambdaClosureType lambda(x);

// 引用捕获
int y = 10;
auto lambda2 = [&y]() { return y; };

// 等价于：
class LambdaClosureType2 {
public:
    LambdaClosureType2(int& captured_y) : y_(captured_y) {}
    
    int operator()() const {
        return y_;
    }
    
private:
    int& y_;
};

LambdaClosureType2 lambda2(y);
```

### 10.3 mutable的实现

```cpp
// mutable Lambda
int z = 10;
auto lambda3 = [z]() mutable {
    z = 20;
    return z;
};

// 等价于：
class LambdaClosureType3 {
public:
    LambdaClosureType3(int captured_z) : z_(captured_z) {}
    
    int operator()() {  // 注意：不再是const
        z_ = 20;
        return z_;
    }
    
private:
    int z_;
};

LambdaClosureType3 lambda3(z);
```

## 十一、Lambda表达式与std::function

### 11.1 转换为std::function

```cpp
#include <functional>

auto lambda = [](int x) { return x * 2; };

// 转换为std::function
std::function<int(int)> func = lambda;

// 使用
std::cout << func(5);  // 10

// 存储在容器中
std::vector<std::function<int(int)>> functions;
functions.push_back([](int x) { return x * 2; });
functions.push_back([](int x) { return x * 3; });

for (auto& f : functions) {
    std::cout << f(5) << " ";
}
```

### 11.2 性能考虑

```cpp
// Lambda直接调用（零开销）
auto lambda = [](int x) { return x * 2; };
int result1 = lambda(5);

// std::function调用（有开销）
std::function<int(int)> func = lambda;
int result2 = func(5);

// 模板参数（零开销）
template<typename F>
int call(F f, int x) {
    return f(x);
}

int result3 = call(lambda, 5);
```

## 十二、Lambda表达式的陷阱

### 12.1 悬空引用

```cpp
// 危险：捕获引用可能悬空
std::function<int()> create_dangling() {
    int x = 10;
    return [&x]() { return x; };  // x在函数返回后销毁
}

// 错误：返回后调用
auto func = create_dangling();
int result = func();  // 未定义行为！

// 正确：按值捕获
std::function<int()> create_safe() {
    int x = 10;
    return [x]() { return x; };  // 按值捕获
}
```

### 12.2 捕获this

```cpp
class MyClass {
public:
    void set_callback() {
        // 捕获this指针
        callback_ = [this]() {
            value_ = 42;
        };
    }
    
    void execute() {
        callback_();
    }
    
private:
    int value_ = 0;
    std::function<void()> callback_;
};

// 危险：对象销毁后this指针悬空
MyClass* obj = new MyClass();
obj->set_callback();
delete obj;
// obj->execute();  // 未定义行为！

// C++17: 捕获*this
auto lambda2 = [*this]() {
    return this->value_;  // 安全：this指向拷贝
};
```

### 12.3 捕获大对象

```cpp
// 按值捕获大对象可能影响性能
std::vector<int> large_data(1000000);

auto lambda = [large_data]() {
    // large_data被拷贝，可能很慢
    return large_data.size();
};

// 更好：按引用捕获或移动捕获
auto lambda2 = [&large_data]() {
    return large_data.size();
};

auto lambda3 = [data = std::move(large_data)]() {
    return data.size();
};
```

### 12.4 mutable的陷阱

```cpp
// mutable Lambda每次调用修改的是同一个拷贝
auto counter = [count = 0]() mutable {
    return ++count;
};

std::cout << counter();  // 1
std::cout << counter();  // 2

// 但是如果是按值捕获外部变量
int external = 0;
auto lambda = [external]() mutable {
    return ++external;
};

std::cout << lambda();  // 1
std::cout << lambda();  // 2
std::cout << external;  // 0（外部变量未改变）
```

## 十三、面试常见问题

### Q1: 什么是Lambda表达式？它有什么优点？

**答案：**
Lambda表达式是一种匿名函数语法，允许在代码中直接定义和使用函数对象。

**优点：**
1. **简洁性**：无需单独定义函数
2. **闭包**：可以捕获外部变量
3. **灵活性**：可以作为参数传递
4. **内联**：编译器可以内联优化
5. **类型安全**：强类型检查

### Q2: 解释Lambda表达式的捕获机制

**答案：**
Lambda表达式可以通过捕获列表捕获外部变量：

1. **按值捕获**：`[x]` - 捕获变量的副本
2. **按引用捕获**：`[&x]` - 捕获变量的引用
3. **隐式捕获**：`[=]` - 按值捕获所有，`[&]` - 按引用捕获所有
4. **混合捕获**：`[=, &x]` - x按引用，其他按值
5. **初始化捕获**：`[x = expr]` - 捕获表达式的结果

### Q3: mutable关键字的作用是什么？

**答案：**
mutable关键字允许Lambda的函数调用运算符是非const的，从而可以修改按值捕获的变量。

```cpp
int x = 10;
auto lambda = [x]() mutable {
    x = 20;  // 修改的是拷贝
    return x;
};
```

### Q4: 泛型Lambda和模板Lambda有什么区别？

**答案：**
- **泛型Lambda (C++14)**：使用`auto`参数，类型推导
- **模板Lambda (C++20)**：使用模板语法，可以约束类型

```cpp
// 泛型Lambda
auto lambda1 = [](auto x) { return x * 2; };

// 模板Lambda
auto lambda2 = []<typename T>(T x) { return x * 2; };

// 模板Lambda可以约束
auto lambda3 = []<std::integral T>(T x) { return x * 2; };
```

### Q5: constexpr Lambda有什么特点？

**答案：**
constexpr Lambda (C++17) 可以在编译时求值，用于常量表达式：

```cpp
constexpr auto square = [](int x) {
    return x * x;
};

constexpr int result = square(5);  // 编译时计算
static_assert(square(5) == 25);
```

### Q6: Lambda表达式和std::function有什么区别？

**答案：**
| 特性 | Lambda | std::function |
|------|--------|---------------|
| 类型 | 独一无二的闭包类型 | 类型擦除 |
| 性能 | 零开销（可内联） | 有虚函数调用开销 |
| 存储 | 可以内联 | 需要堆分配（可能） |
| 转换 | 可转换为std::function | 不能转换为Lambda |

### Q7: 如何在Lambda中实现完美转发？

**答案：**
```cpp
auto perfect_forward = [](auto&&... args) {
    return some_function(std::forward<decltype(args)>(args)...);
};
```

### Q8: Lambda表达式的底层实现是什么？

**答案：**
Lambda表达式编译器会生成一个唯一的闭包类，包含：
1. 捕获的变量作为成员
2. 重载`operator()`
3. 根据捕获方式决定是否为const

```cpp
auto lambda = [x](int y) { return x + y; };

// 等价于：
class ClosureType {
    int x_;
public:
    ClosureType(int x) : x_(x) {}
    int operator()(int y) const { return x_ + y; }
};
```

### Q9: 如何避免Lambda中的悬空引用？

**答案：**
1. 优先按值捕获
2. 如果需要修改，使用mutable
3. 使用智能指针
4. C++17使用`*this`捕获

```cpp
// 安全捕获
auto lambda = [ptr = std::make_shared<int>(42)]() {
    return *ptr;
};
```

### Q10: Lambda表达式在STL算法中的应用

**答案：**
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// 排序
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });

// 查找
auto it = std::find_if(v.begin(), v.end(), [](int x) { return x > 3; });

// 转换
std::transform(v.begin(), v.end(), v.begin(), [](int x) { return x * 2; });
```

## 十四、最佳实践

### 14.1 优先使用Lambda

```cpp
// 好的实践
std::sort(v.begin(), v.end(), [](int a, int b) { return a < b; });

// 避免：单独定义函数
bool compare(int a, int b) { return a < b; }
std::sort(v.begin(), v.end(), compare);
```

### 14.2 避免复杂Lambda

```cpp
// 避免：过于复杂的Lambda
auto lambda = [](int x, int y, int z) {
    // 大量逻辑
    if (x > 0) {
        if (y > 0) {
            if (z > 0) {
                // ...
            }
        }
    }
};

// 更好：提取为函数
int complex_logic(int x, int y, int z);
auto lambda2 = [](int x, int y, int z) {
    return complex_logic(x, y, z);
};
```

### 14.3 使用auto捕获

```cpp
// C++14: 使用初始化捕获
auto lambda = [x = expensive_computation()]() {
    return x;
};

// 避免重复计算
```

### 14.4 注意捕获大小

```cpp
// 避免捕获大对象
std::vector<int> large(1000000);
auto lambda = [&large]() { return large.size(); };  // 按引用捕获
```

### 14.5 使用const Lambda

```cpp
// 优先使用const Lambda（默认）
auto lambda = [](int x) { return x * 2; };  // const

// 只在需要时使用mutable
auto mutable_lambda = [count = 0]() mutable {
    return ++count;
};
```

## 十五、总结

Lambda表达式是现代C++的重要特性，提供了简洁、强大的函数对象创建方式：

**核心特性：**
1. 匿名函数
2. 闭包捕获
3. 类型推导
4. 内联优化

**演进历程：**
- C++11: 基础Lambda
- C++14: 泛型Lambda、初始化捕获
- C++17: constexpr Lambda
- C++20: 模板Lambda

**应用场景：**
1. STL算法
2. 回调函数
3. 事件处理
4. 函数组合

**注意事项：**
1. 避免悬空引用
2. 注意捕获开销
3. 合理使用mutable
4. 保持Lambda简洁

掌握Lambda表达式是现代C++开发的基本技能，也是面试中的高频考点。