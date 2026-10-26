# C++语法规则大全

## 目录
1. [基础语法](#一基础语法)
2. [变量与数据类型](#二变量与数据类型)
3. [运算符](#三运算符)
4. [控制流](#四控制流)
5. [函数](#五函数)
6. [数组与字符串](#六数组与字符串)
7. [指针与引用](#七指针与引用)
8. [结构体与联合体](#八结构体与联合体)
9. [类与对象](#九类与对象)
10. [继承与多态](#十继承与多态)
11. [模板](#十一模板)
12. [异常处理](#十二异常处理)
13. [命名空间](#十三命名空间)
14. [预处理器](#十四预处理器)
15. [IO流](#十五io流)

---

## 一、基础语法

### 1.1 程序结构

```cpp
// 单文件程序
#include <iostream>

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}

// 完整程序结构
#include <iostream>  // 头文件包含

// 全局变量
int global_var = 100;

// 函数声明
void function_declaration();

// 命名空间
namespace my_namespace {
    int namespace_var = 200;
}

// 主函数
int main() {
    // 局部变量
    int local_var = 300;
    
    // 函数调用
    std::cout << "Hello, World!" << std::endl;
    
    return 0;  // 返回语句
}

// 函数定义
void function_declaration() {
    // 函数体
}
```

### 1.2 注释

```cpp
// 单行注释

/*
 * 多行注释
 * 可以跨越多行
 */

/**
 * 文档注释
 * 通常用于函数或类的说明
 */
```

### 1.3 标识符命名规则

```cpp
// 有效标识符
int variable_name;
int _underscore;
int camelCase;
int PascalCase;
int snake_case;
int number123;

// 无效标识符
// int 123number;  // 不能以数字开头
// int my-var;     // 不能包含连字符
// int class;      // 不能是关键字
// int my name;    // 不能包含空格
```

---

## 二、变量与数据类型

### 2.1 基本数据类型

```cpp
// 整数类型
int integer = 42;               // 通常4字节
short short_int = 10;           // 2字节
long long_int = 100L;           // 4或8字节
long long very_long = 100LL;    // 8字节

// 无符号整数
unsigned int unsigned_int = 42;
unsigned short unsigned_short = 10;
unsigned long unsigned_long = 100L;
unsigned long long unsigned_long_long = 100LL;

// 浮点类型
float float_num = 3.14f;       // 单精度，4字节
double double_num = 3.14159;    // 双精度，8字节
long double long_double = 3.14159L;  // 扩展精度

// 字符类型
char char_var = 'A';            // 1字节
wchar_t wchar_var = L'A';       // 宽字符
char16_t char16_var = u'A';     // UTF-16
char32_t char32_var = U'A';     // UTF-32

// 布尔类型
bool bool_var = true;           // true或false
bool bool_var2 = false;

// 空类型
void void_type;                 // 表示无类型

// 空指针
nullptr_t null_ptr = nullptr;   // C++11
```

### 2.2 类型修饰符

```cpp
// const：常量
const int constant = 42;
// constant = 100;  // 错误：不能修改

// volatile：易变
volatile int volatile_var = 10;  // 告诉编译器不要优化

// mutable：可变成员
class MyClass {
    mutable int mutable_var = 0;
public:
    void modify() const {
        mutable_var = 10;  // const函数中可以修改mutable成员
    }
};

// signed/unsigned：有符号/无符号
signed int signed_int = -10;
unsigned int unsigned_int = 10;

// short/long：大小修饰
short short_var = 10;
long long_var = 10;
long long long_long_var = 10;
```

### 2.3 变量声明与定义

```cpp
// 声明（不分配内存）
extern int extern_var;

// 定义（分配内存）
int defined_var = 42;

// 初始化
int initialized = 100;
int initialized_with_list = {200};
int zero_initialized{};  // 零初始化

// 多变量声明
int a = 1, b = 2, c = 3;

// 结构化绑定（C++17）
std::pair<int, double> pair = {42, 3.14};
auto [x, y] = pair;
```

### 2.4 类型推导

```cpp
// auto类型推导（C++11）
auto auto_var = 42;                    // int
auto auto_var2 = 3.14;                 // double
auto auto_var3 = "hello";              // const char*

// decltype类型推导（C++11）
int x = 10;
decltype(x) y = 20;                    // int

// decltype(auto)（C++14）
decltype(auto) z = x;                  // int&

// 推导指引（C++17）
std::vector vec{1, 2, 3};  // std::vector<int>
```

### 2.5 类型转换

```cpp
// 隐式转换
int int_var = 10;
double double_var = int_var;  // int转double

// C风格转换（不推荐）
double d = 3.14;
int i = (int)d;

// C++风格转换（推荐）
// static_cast：相关类型转换
double d1 = 3.14;
int i1 = static_cast<int>(d1);

// const_cast：移除const属性
const int ci = 10;
int* pi = const_cast<int*>(&ci);

// dynamic_cast：类层次转换
Base* base_ptr = new Derived();
Derived* derived_ptr = dynamic_cast<Derived*>(base_ptr);

// reinterpret_cast：不相关类型转换
int* ip = reinterpret_cast<int*>(0x12345678);

// C++20：std::bit_cast
float f = 3.14f;
int i = std::bit_cast<int>(f);
```

---

## 三、运算符

### 3.1 算术运算符

```cpp
int a = 10, b = 3;

// 加减乘除模
int add = a + b;      // 13
int sub = a - b;      // 7
int mul = a * b;      // 30
int div = a / b;      // 3（整数除法）
int mod = a % b;      // 1（取模）

// 自增自减
int x = 10;
int pre_increment = ++x;  // x=11, pre_increment=11
int post_increment = x++; // x=12, post_increment=11
int pre_decrement = --x;  // x=11, pre_decrement=11
int post_decrement = x--; // x=10, post_decrement=11
```

### 3.2 关系运算符

```cpp
int a = 10, b = 20;

bool equal = (a == b);    // false
bool not_equal = (a != b); // true
bool less = (a < b);       // true
bool less_equal = (a <= b); // true
bool greater = (a > b);    // false
bool greater_equal = (a >= b); // false

// 三相比较（C++20）
auto cmp = a <=> b;  // std::strong_ordering::less
```

### 3.3 逻辑运算符

```cpp
bool a = true, b = false;

bool logical_and = a && b;  // false
bool logical_or = a || b;   // true
bool logical_not = !a;      // false

// 短路求值
int x = 0, y = 10;
bool result = (x != 0) && (y / x > 0);  // 安全，不会除零
```

### 3.4 位运算符

```cpp
unsigned int a = 0b1010;  // 10
unsigned int b = 0b1100;  // 12

// 位运算
unsigned int bit_and = a & b;    // 0b1000 (8)
unsigned int bit_or = a | b;     // 0b1110 (14)
unsigned int bit_xor = a ^ b;    // 0b0110 (6)
unsigned int bit_not = ~a;       // 0b...11110101

// 移位
unsigned int left_shift = a << 2;   // 0b101000 (40)
unsigned int right_shift = a >> 2;  // 0b10 (2)
```

### 3.5 赋值运算符

```cpp
int a = 10;
int b = 20;

// 简单赋值
a = b;

// 复合赋值
a += b;   // a = a + b
a -= b;   // a = a - b
a *= b;   // a = a * b
a /= b;   // a = a / b
a %= b;   // a = a % b
a &= b;   // a = a & b
a |= b;   // a = a | b
a ^= b;   // a = a ^ b
a <<= b;  // a = a << b
a >>= b;  // a = a >> b
```

### 3.6 其他运算符

```cpp
// 条件运算符
int a = 10, b = 20;
int max = (a > b) ? a : b;  // 20

// 逗号运算符
int x = (a++, b++, a + b);  // 33

// sizeof运算符
int size_int = sizeof(int);           // 类型大小
int size_var = sizeof(a);              // 变量大小
int size_expr = sizeof(a + b);         // 表达式大小

// alignof运算符（C++11）
int align_int = alignof(int);

// typeid运算符（运行时类型信息）
#include <typeinfo>
std::cout << typeid(a).name() << std::endl;

// 成员访问运算符
struct Point { int x, y; };
Point p = {1, 2};
int x1 = p.x;           // 直接成员访问
Point* ptr = &p;
int x2 = ptr->x;        // 指针成员访问

// 成员指针访问
int Point::* member_ptr = &Point::x;
int x3 = p.*member_ptr;     // 对象成员指针
int x4 = (ptr->*member_ptr); // 指针成员指针

// 作用域解析运算符
int global_var = 100;
int x = ::global_var;  // 访问全局变量

namespace NS {
    int var = 200;
}
int y = NS::var;  // 访问命名空间变量

// 空指针常量（C++11）
int* ptr = nullptr;

// new/delete运算符
int* dynamic_ptr = new int(42);
delete dynamic_ptr;

int* array_ptr = new int[10];
delete[] array_ptr;
```

### 3.7 运算符优先级

```cpp
// 从高到低
// 1. 作用域解析 ::
// 2. 后缀 ++ -- () [] . -> ++ --
// 3. 一元 + - ! ~ ++ -- sizeof new delete
// 4. 成员指针 .* ->*
// 5. 乘除模 * / %
// 6. 加减 + -
// 7. 移位 << >>
// 8. 关系 < <= > >=
// 9. 相等 == !=
// 10. 位与 &
// 11. 位异或 ^
// 12. 位或 |
// 13. 逻辑 &&
// 14. 逻辑 ||
// 15. 条件 ?:
// 16. 赋值 = += -= *= /= %= &= |= ^= <<= >>=
// 17. 逗号 ,

// 使用括号明确优先级
int result = (a + b) * c;
```

---

## 四、控制流

### 4.1 if语句

```cpp
// 基本if
if (condition) {
    // 条件为真时执行
}

// if-else
if (condition) {
    // 条件为真时执行
} else {
    // 条件为假时执行
}

// if-else if-else
if (condition1) {
    // 条件1为真时执行
} else if (condition2) {
    // 条件2为真时执行
} else {
    // 所有条件都为假时执行
}

// 嵌套if
if (outer_condition) {
    if (inner_condition) {
        // 内部条件为真时执行
    }
}

// if初始化语句（C++17）
if (auto result = calculate(); result > 0) {
    // 使用result
}
```

### 4.2 switch语句

```cpp
// 基本switch
int value = 2;
switch (value) {
    case 1:
        // value等于1时执行
        break;
    case 2:
        // value等于2时执行
        break;
    default:
        // 所有case都不匹配时执行
        break;
}

// fallthrough（故意穿透）
switch (value) {
    case 1:
        do_something();
        [[fallthrough]];  // C++17显式标记fallthrough
    case 2:
        do_something_else();
        break;
}

// switch初始化语句（C++17）
switch (int x = get_value(); x) {
    case 1:
        break;
    default:
        break;
}
```

### 4.3 循环语句

```cpp
// while循环
int i = 0;
while (i < 10) {
    // 循环体
    ++i;
}

// do-while循环
int j = 0;
do {
    // 循环体
    ++j;
} while (j < 10);

// for循环
for (int k = 0; k < 10; ++k) {
    // 循环体
}

// 范围for循环（C++11）
std::vector<int> vec = {1, 2, 3, 4, 5};
for (int elem : vec) {
    std::cout << elem << std::endl;
}

// 范围for循环（引用）
for (int& elem : vec) {
    elem *= 2;  // 修改元素
}

// 基于迭代器的范围for
for (auto it = vec.begin(); it != vec.end(); ++it) {
    std::cout << *it << std::endl;
}
```

### 4.4 跳转语句

```cpp
// break：跳出循环或switch
for (int i = 0; i < 10; ++i) {
    if (i == 5) {
        break;  // 跳出循环
    }
}

// continue：跳过本次循环
for (int i = 0; i < 10; ++i) {
    if (i == 5) {
        continue;  // 跳过i=5
    }
    std::cout << i << std::endl;
}

// goto：跳转到标签（不推荐）
start:
    // 代码
    if (condition) {
        goto start;  // 跳转到start标签
    }

// return：返回函数
int function() {
    return 42;
}

// 协程关键字（C++20）
generator<int> my_generator() {
    co_yield 1;
    co_yield 2;
    co_return;
}
```

### 4.5 if constexpr（C++17）

```cpp
// 编译时条件分支
template<typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 2.0;
    } else {
        return value;
    }
}
```

---

## 五、函数

### 5.1 函数定义

```cpp
// 基本函数定义
返回类型 函数名(参数列表) {
    // 函数体
    return 返回值;
}

// 示例
int add(int a, int b) {
    return a + b;
}

// 无返回值
void print_hello() {
    std::cout << "Hello, World!" << std::endl;
}

// 无参数
void no_params() {
    std::cout << "No parameters" << std::endl;
}
```

### 5.2 函数声明

```cpp
// 函数声明（原型）
int add(int a, int b);
void print_hello();

// 在声明中可以省略参数名
int add(int, int);

// inline函数
inline int multiply(int a, int b) {
    return a * b;
}

// constexpr函数（C++11）
constexpr int square(int x) {
    return x * x;
}
constexpr int result = square(5);  // 编译时计算
```

### 5.3 参数传递

```cpp
// 值传递
void by_value(int x) {
    x = 100;  // 修改不影响外部
}

// 指针传递
void by_pointer(int* ptr) {
    *ptr = 100;  // 修改影响外部
}

// 引用传递
void by_reference(int& ref) {
    ref = 100;  // 修改影响外部
}

// const引用传递（避免拷贝）
void by_const_reference(const std::string& str) {
    std::cout << str << std::endl;
}

// 右值引用传递（C++11）
void by_rvalue_reference(std::string&& str) {
    std::string moved = std::move(str);  // 移动语义
}

// 默认参数
int default_params(int a, int b = 10, int c = 20) {
    return a + b + c;
}

// 可变参数（C风格）
void variadic_c_style(int count, ...) {
    va_list args;
    va_start(args, count);
    for (int i = 0; i < count; ++i) {
        int value = va_arg(args, int);
        std::cout << value << std::endl;
    }
    va_end(args);
}

// 可变参数模板（C++11）
template<typename... Args>
void variadic_template(Args... args) {
    (std::cout << ... << args) << std::endl;  // C++17折叠表达式
}
```

### 5.4 函数重载

```cpp
// 函数重载：同名函数，不同参数
int add(int a, int b) {
    return a + b;
}

double add(double a, double b) {
    return a + b;
}

std::string add(const std::string& a, const std::string& b) {
    return a + b;
}

// 使用
int i = add(10, 20);              // 调用int版本
double d = add(1.5, 2.5);         // 调用double版本
std::string s = add("Hello", " World");  // 调用string版本
```

### 5.5 函数指针

```cpp
// 函数指针类型
int (*func_ptr)(int, int);

// 函数指针赋值
func_ptr = add;
int result = func_ptr(10, 20);

// 函数指针数组
int (*func_array[3])(int, int);
func_array[0] = add;
func_array[1] = subtract;
func_array[2] = multiply;

// 使用typedef定义函数指针
typedef int (*FuncType)(int, int);
FuncType func_ptr2 = add;

// 使用using定义函数指针（C++11）
using FuncType2 = int(*)(int, int);
FuncType2 func_ptr3 = add;
```

### 5.6 lambda表达式（C++11）

```cpp
// 基本lambda
auto lambda = [](int x, int y) {
    return x + y;
};

// 捕获列表
int a = 10, b = 20;
auto lambda1 = [a, b](int x) {  // 按值捕获
    return x + a + b;
};

auto lambda2 = [&a, &b](int x) {  // 按引用捕获
    return x + a + b;
};

auto lambda3 = [=](int x) {  // 按值捕获所有
    return x + a + b;
};

auto lambda4 = [&](int x) {  // 按引用捕获所有
    return x + a + b;
};

auto lambda5 = [=, &b](int x) {  // b按引用，其他按值
    return x + a + b;
};

// mutable lambda
auto lambda6 = [a]() mutable {
    a = 100;  // 可以修改捕获的副本
    return a;
};

// 尾返回类型
auto lambda7 = [](int x, int y) -> int {
    return x + y;
};

// 泛型lambda（C++14）
auto lambda8 = [](auto x, auto y) {
    return x + y;
};

// constexpr lambda（C++17）
constexpr auto lambda9 = [](int x) {
    return x * 2;
};

// 模板lambda（C++20）
auto lambda10 = []<typename T>(T x) {
    return x * 2;
};
```

### 5.7 函数对象（仿函数）

```cpp
// 函数对象类
struct Add {
    int operator()(int a, int b) const {
        return a + b;
    }
};

// 使用
Add add_func;
int result = add_func(10, 20);
```

### 5.8 std::function（C++11）

```cpp
#include <functional>

// 存储函数指针
std::function<int(int, int)> func1 = add;

// 存储lambda
std::function<int(int, int)> func2 = [](int a, int b) {
    return a + b;
};

// 存储函数对象
std::function<int(int, int)> func3 = Add();

// 存储
std::vector<std::function<int(int, int)>> functions;
functions.push_back(add);
functions.push_back([](int a, int b) { return a + b; });
functions.push_back(Add());
```

### 5.9 std::bind（C++11）

```cpp
#include <functional>

// 绑定函数参数
int multiply(int a, int b) {
    return a * b;
}

// 绑定第一个参数
auto multiply_by_10 = std::bind(multiply, 10, std::placeholders::_1);
int result1 = multiply_by_10(5);  // 50

// 绑定第二个参数
auto multiply_10_by = std::bind(multiply, std::placeholders::_1, 10);
int result2 = multiply_10_by(5);  // 50

// 绑定所有参数
auto multiply_10_by_5 = std::bind(multiply, 10, 5);
int result3 = multiply_10_by_5();  // 50

// 绑定成员函数
class MyClass {
public:
    int add(int a, int b) { return a + b; }
};

MyClass obj;
auto member_func = std::bind(&MyClass::add, &obj, std::placeholders::_1, std::placeholders::_2);
int result4 = member_func(10, 20);  // 30
```

---

## 六、数组与字符串

### 6.1 数组

```cpp
// 静态数组
int arr1[5];                     // 未初始化
int arr2[5] = {1, 2, 3, 4, 5};  // 初始化
int arr3[] = {1, 2, 3, 4, 5};    // 自动推导大小
int arr4[5] = {1, 2};             // 部分初始化，其余为0

// 访问数组元素
int value = arr2[0];  // 第一个元素
arr2[1] = 10;         // 修改第二个元素

// 数组遍历
for (int i = 0; i < 5; ++i) {
    std::cout << arr2[i] << std::endl;
}

// 范围for遍历
for (int elem : arr2) {
    std::cout << elem << std::endl;
}

// 多维数组
int multi_arr[3][4] = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12}
};

// 访问多维数组
int value = multi_arr[1][2];  // 7

// 动态数组
int* dynamic_arr = new int[5];
dynamic_arr[0] = 1;
delete[] dynamic_arr;

// std::array（C++11）
#include <array>
std::array<int, 5> std_arr = {1, 2, 3, 4, 5};
std::cout << std_arr.size() << std::endl;  // 5
```

### 6.2 字符串

```cpp
// C风格字符串
char c_str1[] = "Hello";           // 自动添加'\0'
char c_str2[6] = {'H', 'e', 'l', 'l', 'o', '\0'};
const char* c_str3 = "Hello";

// C字符串函数
#include <cstring>
size_t len = strlen(c_str1);           // 长度
char* copy = strdup(c_str1);            // 复制
int cmp = strcmp(c_str1, "Hello");      // 比较
char* cat = strcat(c_str1, " World");   // 连接

// std::string
#include <string>
std::string str1 = "Hello";
std::string str2 = " World";

// 字符串操作
std::string str3 = str1 + str2;           // 连接
bool equal = (str1 == "Hello");           // 比较
size_t size = str1.size();                // 大小
char ch = str1[0];                        // 访问字符
str1[0] = 'h';                            // 修改字符

// 成员函数
str1.append(" World");                     // 追加
str1.push_back('!');                       // 追加字符
size_t pos = str1.find("World");           // 查找
std::string sub = str1.substr(0, 5);       // 子串
str1.replace(6, 5, "C++");                // 替换
str1.erase(5, 6);                          // 删除

// 转换
const char* c_str = str1.c_str();         // 转为C字符串
int num = std::stoi("42");                 // 字符串转整数
std::string num_str = std::to_string(42);  // 整数转字符串

// 字符串视图（C++17）
#include <string_view>
std::string_view sv = "Hello";  // 不拷贝
```

### 6.3 std::vector

```cpp
#include <vector>

// 创建vector
std::vector<int> vec1;                       // 空
std::vector<int> vec2(5);                     // 5个元素，默认为0
std::vector<int> vec3(5, 100);               // 5个元素，全部为100
std::vector<int> vec4 = {1, 2, 3, 4, 5};     // 初始化列表

// 添加元素
vec1.push_back(10);                          // 尾部添加
vec1.emplace_back(20);                       // 尾部构造
vec1.insert(vec1.begin(), 5);                // 插入

// 访问元素
int first = vec1.front();                     // 第一个元素
int last = vec1.back();                      // 最后一个元素
int element = vec1[0];                       // 索引访问
int element2 = vec1.at(0);                   // 带边界检查

// 大小和容量
size_t size = vec1.size();                   // 元素个数
size_t capacity = vec1.capacity();           // 容量
bool empty = vec1.empty();                   // 是否为空
vec1.resize(10);                             // 调整大小
vec1.reserve(20);                            // 预留容量

// 删除元素
vec1.pop_back();                             // 删除最后一个
vec1.erase(vec1.begin());                    // 删除指定位置
vec1.clear();                                // 清空

// 遍历
for (int elem : vec1) {
    std::cout << elem << std::endl;
}

for (auto it = vec1.begin(); it != vec1.end(); ++it) {
    std::cout << *it << std::endl;
}
```

---

## 七、指针与引用

### 7.1 指针

```cpp
// 指针声明
int value = 42;
int* ptr = &value;          // 指向value的指针

// 解引用
int deref = *ptr;           // 42

// 修改指向的值
*ptr = 100;                // value变为100

// 指针运算
int arr[5] = {1, 2, 3, 4, 5};
int* arr_ptr = arr;
int first = *arr_ptr;       // 1
int second = *(arr_ptr + 1); // 2

// 指针比较
int* ptr1 = &value;
int* ptr2 = &value;
bool equal = (ptr1 == ptr2); // true

// 空指针
int* null_ptr = nullptr;    // C++11
int* null_ptr2 = NULL;      // C风格
int* null_ptr3 = 0;         // 整数0

// void指针
void* void_ptr = &value;
int* int_ptr = static_cast<int*>(void_ptr);

// const指针
int const* ptr_to_const = &value;   // 指向常量的指针
int* const const_ptr = &value;      // 常量指针
int const* const const_ptr_to_const = &value;  // 指向常量的常量指针

// 指针的指针
int** ptr_to_ptr = &ptr;

// 函数指针
int (*func_ptr)(int, int) = add;
int result = func_ptr(10, 20);

// 智能指针（C++11）
#include <memory>
std::unique_ptr<int> unique_ptr = std::make_unique<int>(42);
std::shared_ptr<int> shared_ptr = std::make_shared<int>(42);
std::weak_ptr<int> weak_ptr = shared_ptr;
```

### 7.2 引用

```cpp
// 引用声明
int value = 42;
int& ref = value;          // value的引用

// 通过引用修改
ref = 100;                // value变为100

// 引用作为参数
void by_reference(int& ref) {
    ref = 100;            // 修改外部变量
}

// const引用
void by_const_reference(const int& ref) {
    // ref = 100;        // 错误：不能修改
    int value = ref;     // 可以读取
}

// 引用作为返回值
int& get_reference(int& value) {
    return value;        // 返回引用
}

// 引用与指针的区别
int a = 10;
int* ptr = &a;          // 指针，可以重新赋值
int& ref = a;           // 引用，初始化后不能重新赋值

int b = 20;
ptr = &b;              // 指针可以指向其他变量
// ref = b;            // 错误：引用不能重新赋值
```

### 7.3 右值引用（C++11）

```cpp
// 右值引用
int&& rref1 = 10;                       // 绑定到字面量
int&& rref2 = std::move(value);         // 将左值转换为右值引用

// 完美转发
template<typename T>
void perfect_forward(T&& arg) {
    target(std::forward<T>(arg));
}
```

---

## 八、结构体与联合体

### 8.1 结构体

```cpp
// 结构体定义
struct Point {
    int x;
    int y;
    
    // 成员函数
    void print() const {
        std::cout << "(" << x << ", " << y << ")" << std::endl;
    }
};

// 结构体初始化
Point p1 = {1, 2};
Point p2{3, 4};  // C++11统一初始化

// 访问成员
int x = p1.x;
p1.y = 10;

// 结构体指针
Point* ptr = &p1;
int x2 = ptr->x;
ptr->y = 20;

// 结构体数组
Point points[3] = {{1, 2}, {3, 4}, {5, 6}};

// 结构体作为参数
void print_point(Point p) {
    std::cout << "(" << p.x << ", " << p.y << ")" << std::endl;
}

// 结构体作为返回值
Point create_point(int x, int y) {
    return {x, y};
}

// 位域
struct BitField {
    unsigned int a : 3;  // 3位
    unsigned int b : 5;  // 5位
    unsigned int c : 8;  // 8位
};
```

### 8.2 联合体

```cpp
// 联合体定义
union Data {
    int i;
    float f;
    char str[20];
};

// 联合体初始化
Data data;
data.i = 42;            // 存储整数
std::cout << data.i << std::endl;

data.f = 3.14f;         // 存储浮点数
std::cout << data.f << std::endl;

// 匿名联合体
struct MyStruct {
    int type;
    union {
        int i;
        float f;
        char str[20];
    };
};

// C++11联合体改进
union ModernUnion {
    int i;
    float f;
    std::string str;    // 可以有非POD类型
    
    ModernUnion() {}     // 需要构造函数
    ~ModernUnion() {}    // 需要析构函数
};
```

### 8.3 枚举

```cpp
// C风格枚举
enum Color {
    RED,
    GREEN,
    BLUE
};

Color c = RED;

// 指定值
enum Size {
    SMALL = 1,
    MEDIUM = 2,
    LARGE = 3
};

// C++11强类型枚举
enum class Color2 {
    RED,
    GREEN,
    BLUE
};

Color2 c2 = Color2::RED;  // 需要限定符

// 指定底层类型
enum class Color3 : uint8_t {
    RED = 0,
    GREEN = 1,
    BLUE = 2
};
```

---

## 九、类与对象

### 9.1 类定义

```cpp
// 类定义
class MyClass {
public:                     // 公有成员
    // 构造函数
    MyClass() : value_(0) {}
    
    MyClass(int value) : value_(value) {}
    
    // 析构函数
    ~MyClass() {}
    
    // 成员函数
    int get_value() const {
        return value_;
    }
    
    void set_value(int value) {
        value_ = value;
    }
    
    // 静态成员函数
    static int static_func() {
        return 42;
    }
    
protected:                  // 保护成员
    int protected_value_;
    
private:                    // 私有成员
    int value_;
    
    // 静态成员变量
    static int static_value_;
};

// 静态成员变量定义
int MyClass::static_value_ = 100;

// 使用类
MyClass obj;
obj.set_value(42);
int value = obj.get_value();
```

### 9.2 构造函数

```cpp
class MyClass {
public:
    // 默认构造函数
    MyClass() : value_(0) {}
    
    // 参数化构造函数
    MyClass(int value) : value_(value) {}
    
    // 拷贝构造函数
    MyClass(const MyClass& other) : value_(other.value_) {}
    
    // 移动构造函数（C++11）
    MyClass(MyClass&& other) noexcept : value_(other.value_) {
        other.value_ = 0;
    }
    
    // 拷贝赋值运算符
    MyClass& operator=(const MyClass& other) {
        if (this != &other) {
            value_ = other.value_;
        }
        return *this;
    }
    
    // 移动赋值运算符（C++11）
    MyClass& operator=(MyClass&& other) noexcept {
        if (this != &other) {
            value_ = other.value_;
            other.value_ = 0;
        }
        return *this;
    }
    
    // 委托构造函数（C++11）
    MyClass(int a, int b) : MyClass(a + b) {}
    
private:
    int value_;
};
```

### 9.3 析构函数

```cpp
class MyClass {
public:
    // 构造函数
    MyClass() : data_(new int[100]) {
        std::cout << "Constructor" << std::endl;
    }
    
    // 析构函数
    ~MyClass() {
        std::cout << "Destructor" << std::endl;
        delete[] data_;
    }
    
private:
    int* data_;
};
```

### 9.4 const成员函数

```cpp
class MyClass {
public:
    // const成员函数
    int get_value() const {
        return value_;  // 可以读取
        // value_ = 100;  // 错误：不能修改
    }
    
    // mutable成员
    mutable int counter_ = 0;
    
    void increment() const {
        counter_++;  // 可以修改mutable成员
    }
    
private:
    int value_ = 0;
};
```

### 9.5 静态成员

```cpp
class MyClass {
public:
    // 静态成员变量
    static int static_value;
    
    // 静态成员函数
    static int static_function() {
        return static_value;
    }
};

// 静态成员变量定义
int MyClass::static_value = 100;

// 使用
int value = MyClass::static_function();
```

### 9.6 友元

```cpp
class MyClass {
private:
    int private_value_ = 42;
    
    // 友元函数
    friend void friend_function(MyClass& obj);
    
    // 友元类
    friend class FriendClass;
};

void friend_function(MyClass& obj) {
    obj.private_value_ = 100;  // 可以访问私有成员
}

class FriendClass {
public:
    void access(MyClass& obj) {
        obj.private_value_ = 200;  // 可以访问私有成员
    }
};
```

### 9.7 运算符重载

```cpp
class Complex {
public:
    double real, imag;
    
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}
    
    // 成员函数重载
    Complex operator+(const Complex& other) const {
        return Complex(real + other.real, imag + other.imag);
    }
    
    // 赋值运算符
    Complex& operator=(const Complex& other) {
        if (this != &other) {
            real = other.real;
            imag = other.imag;
        }
        return *this;
    }
    
    // 下标运算符
    double& operator[](int index) {
        return (index == 0) ? real : imag;
    }
    
    // 函数调用运算符
    double operator()() const {
        return std::sqrt(real * real + imag * imag);
    }
    
    // 前置++和后置++
    Complex& operator++() {
        ++real;
        return *this;
    }
    
    Complex operator++(int) {
        Complex temp = *this;
        ++real;
        return temp;
    }
};

// 非成员函数重载
Complex operator*(double scalar, const Complex& c) {
    return Complex(scalar * c.real, scalar * c.imag);
}

// 输入输出运算符重载
std::ostream& operator<<(std::ostream& os, const Complex& c) {
    os << c.real << " + " << c.imag << "i";
    return os;
}

std::istream& operator>>(std::istream& is, Complex& c) {
    is >> c.real >> c.imag;
    return is;
}
```

### 9.8 转换运算符

```cpp
class MyClass {
public:
    int value_ = 42;
    
    // 转换为int
    operator int() const {
        return value_;
    }
    
    // 转换为bool（C++11 explicit）
    explicit operator bool() const {
        return value_ != 0;
    }
};

// 使用
MyClass obj;
int i = obj;           // 隐式转换
if (obj) {            // 显式转换
    // ...
}
```

---

## 十、继承与多态

### 10.1 继承

```cpp
// 基类
class Base {
public:
    int base_value_ = 10;
    
    void base_method() {
        std::cout << "Base method" << std::endl;
    }
    
    virtual void virtual_method() {
        std::cout << "Base virtual method" << std::endl;
    }
    
    virtual ~Base() = default;
};

// 公有继承
class Derived : public Base {
public:
    int derived_value_ = 20;
    
    void derived_method() {
        std::cout << "Derived method" << std::endl;
    }
    
    void virtual_method() override {
        std::cout << "Derived virtual method" << std::endl;
    }
};

// 保护继承
class ProtectedDerived : protected Base {
    // base_value_变为protected
};

// 私有继承
class PrivateDerived : private Base {
    // base_value_变为private
};

// 多重继承
class MultipleDerived : public Base, public AnotherBase {
    // ...
};
```

### 10.2 虚函数与多态

```cpp
class Shape {
public:
    virtual double area() const = 0;  // 纯虚函数
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    Circle(double radius) : radius_(radius) {}
    
    double area() const override {
        return 3.14159 * radius_ * radius_;
    }
    
private:
    double radius_;
};

class Rectangle : public Shape {
public:
    Rectangle(double width, double height) 
        : width_(width), height_(height) {}
    
    double area() const override {
        return width_ * height_;
    }
    
private:
    double width_, height_;
};

// 多态使用
Shape* shape1 = new Circle(5.0);
Shape* shape2 = new Rectangle(3.0, 4.0);

std::cout << shape1->area() << std::endl;  // 调用Circle::area
std::cout << shape2->area() << std::endl;  // 调用Rectangle::area

delete shape1;
delete shape2;
```

### 10.3 虚继承

```cpp
class A {
public:
    int value = 10;
};

class B : virtual public A {};
class C : virtual public A {};

class D : public B, public C {
    // 只有一个A的副本
};

D d;
d.value = 20;  // 无二义性
```

### 10.4 override和final（C++11）

```cpp
class Base {
public:
    virtual void func1() {}
    virtual void func2() {}
    virtual void func3() {}
};

class Derived : public Base {
public:
    // override：确保覆盖基类虚函数
    void func1() override {}
    
    // final：禁止进一步覆盖
    void func2() final {}
    
    // 错误：没有对应的基类虚函数
    // void func4() override {}
    
    // 错误：参数不匹配
    // void func1(int x) override {}
};

class FurtherDerived : public Derived {
public:
    void func1() override {}  // OK
    
    // 错误：func2是final
    // void func2() override {}
};

// final类
class FinalClass final {};
// 错误：FinalClass是final
// class DerivedFromFinal : public FinalClass {};
```

---

## 十一、模板

### 11.1 函数模板

```cpp
// 基本函数模板
template<typename T>
T add(T a, T b) {
    return a + b;
}

// 使用
int i = add(10, 20);       // T推导为int
double d = add(1.5, 2.5);  // T推导为double

// 多个模板参数
template<typename T, typename U>
auto multiply(T a, U b) {
    return a * b;
}

// 模板参数推导
template<typename T>
void print_type(T value) {
    std::cout << typeid(T).name() << std::endl;
}
```

### 11.2 类模板

```cpp
// 基本类模板
template<typename T>
class Stack {
public:
    void push(const T& value);
    T pop();
    bool empty() const;
    
private:
    std::vector<T> data_;
};

// 使用
Stack<int> int_stack;
Stack<std::string> string_stack;

// 模板成员函数
template<typename T>
void Stack<T>::push(const T& value) {
    data_.push_back(value);
}

// 模板特化
template<>
class Stack<bool> {
    // bool的特殊实现
};
```

### 11.3 模板特化

```cpp
// 全特化
template<typename T>
class MyClass {
    // 通用版本
};

template<>
class MyClass<int> {
    // int的特殊版本
};

// 偏特化
template<typename T, typename U>
class MyClass2 {
    // 通用版本
};

template<typename T>
class MyClass2<T, int> {
    // 第二个参数是int的特殊版本
};

template<typename T>
class MyClass2<T, T> {
    // 两个参数类型相同
};
```

### 11.4 非类型模板参数

```cpp
// 非类型模板参数
template<int N>
class Array {
public:
    int data_[N];
    
    int size() const { return N; }
};

// 使用
Array<5> arr;

// 常量表达式模板（C++11）
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

constexpr int fact5 = Factorial<5>::value;  // 120
```

### 11.5 变参模板（C++11）

```cpp
// 变参模板
template<typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << std::endl;  // C++17折叠表达式
}

// 使用
print(1, 2.5, "hello", 'a');  // 12.5helloa

// 递归展开（C++11/14）
template<typename First, typename... Rest>
void print_recursive(First first, Rest... rest) {
    std::cout << first << std::endl;
    if constexpr (sizeof...(rest) > 0) {
        print_recursive(rest...);
    }
}

// 参数包大小
template<typename... Args>
constexpr size_t count() {
    return sizeof...(Args);
}

constexpr size_t count1 = count<int, double, std::string>();  // 3
```

### 11.6 SFINAE

```cpp
// enable_if
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
add(T a, T b) {
    return a + b;
}

// C++14 enable_if_t
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
multiply(T a, T b) {
    return a * b;
}

// requires子句（C++20）
template<typename T>
requires std::integral<T>
T divide(T a, T b) {
    return a / b;
}
```

### 11.7 Concepts（C++20）

```cpp
// 定义Concept
template<typename T>
concept Integral = std::is_integral_v<T>;

template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

// 使用Concept
template<Integral T>
T add(T a, T b) {
    return a + b;
}

// requires表达式
template<typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;
    } else {
        return value * 2.0;
    }
}
```

---

## 十二、异常处理

### 12.1 基本异常处理

```cpp
// 抛出异常
void throw_exception() {
    throw std::runtime_error("An error occurred");
}

// 捕获异常
try {
    throw_exception();
} catch (const std::runtime_error& e) {
    std::cout << "Caught runtime_error: " << e.what() << std::endl;
} catch (const std::exception& e) {
    std::cout << "Caught exception: " << e.what() << std::endl;
} catch (...) {
    std::cout << "Caught unknown exception" << std::endl;
}
```

### 12.2 自定义异常

```cpp
// 自定义异常类
class MyException : public std::exception {
public:
    MyException(const std::string& message) : message_(message) {}
    
    const char* what() const noexcept override {
        return message_.c_str();
    }
    
private:
    std::string message_;
};

// 抛出自定义异常
void throw_custom_exception() {
    throw MyException("Custom error message");
}

try {
    throw_custom_exception();
} catch (const MyException& e) {
    std::cout << e.what() << std::endl;
}
```

### 12.3 noexcept（C++11）

```cpp
// noexcept函数
void no_except_function() noexcept {
    // 不会抛出异常
}

// 条件noexcept
void conditional_noexcept() noexcept(noexcept(some_function())) {
    some_function();
}

// noexcept运算符
constexpr bool will_throw = noexcept(no_except_function());  // false
```

---

## 十三、命名空间

### 13.1 命名空间定义

```cpp
// 命名空间定义
namespace MyNamespace {
    int value = 100;
    
    void function() {
        std::cout << "MyNamespace function" << std::endl;
    }
    
    // 嵌套命名空间
    namespace InnerNamespace {
        int inner_value = 200;
    }
}

// 访问命名空间成员
int v = MyNamespace::value;
MyNamespace::function();
int iv = MyNamespace::InnerNamespace::inner_value;
```

### 13.2 using声明

```cpp
// using声明
using MyNamespace::value;
using MyNamespace::function;

// 直接使用
int v = value;
function();

// using指令
using namespace MyNamespace;
int v2 = value;
function();
```

### 13.3 命名空间别名（C++11）

```cpp
// 命名空间别名
namespace Short = MyNamespace::InnerNamespace;

int iv = Short::inner_value;
```

### 13.4 匿名命名空间

```cpp
// 匿名命名空间
namespace {
    int internal_value = 300;
    
    void internal_function() {
        std::cout << "Internal function" << std::endl;
    }
}

// 只在当前文件中可见
int v = internal_value;
```

---

## 十四、预处理器

### 14.1 #include

```cpp
// 系统头文件
#include <iostream>
#include <vector>
#include <string>

// 自定义头文件
#include "myheader.h"

// 条件包含
#ifdef HAVE_FEATURE
#include "feature.h"
#endif
```

### 14.2 #define

```cpp
// 宏定义
#define PI 3.14159265358979323846
#define MAX_SIZE 100

// 宏函数
#define SQUARE(x) ((x) * (x))

// 使用
double area = PI * radius * radius;
int size = MAX_SIZE;
int result = SQUARE(5);  // 25

// 取消宏定义
#undef PI
```

### 14.3 条件编译

```cpp
// #ifdef / #ifndef / #endif
#ifdef DEBUG
    std::cout << "Debug mode" << std::endl;
#endif

#ifndef NDEBUG
    std::cout << "Debug mode" << std::endl;
#endif

// #if / #elif / #else / #endif
#if defined(PLATFORM_WINDOWS)
    // Windows代码
#elif defined(PLATFORM_LINUX)
    // Linux代码
#else
    // 其他平台代码
#endif

// defined运算符
#if defined(__cplusplus) && __cplusplus >= 201703L
    // C++17代码
#endif
```

### 14.4 #error和#warning

```cpp
// #error：编译时错误
#if !defined(VERSION)
    #error "VERSION must be defined"
#endif

// #warning：编译时警告
#if !defined(OPTIONAL_FEATURE)
    #warning "OPTIONAL_FEATURE not defined"
#endif
```

### 14.5 #pragma

```cpp
// #pragma once：防止头文件重复包含
#pragma once

// 等价于
#ifndef MYHEADER_H
#define MYHEADER_H

// 头文件内容

#endif

// #pragma message：编译时消息
#pragma message("Compiling MyHeader.h")

// #pragma warning：控制警告
#pragma warning(push)
#pragma warning(disable: 4996)
void old_function() {}
#pragma warning(pop)
```

### 14.6 static_assert（C++11）

```cpp
// 编译时断言
static_assert(sizeof(int) == 4, "int must be 4 bytes");

// C++17可以省略消息
static_assert(sizeof(int) == 4);
```

### 14.7 __has_include（C++17）

```cpp
// 检查头文件是否存在
#if __has_include(<optional>)
    #include <optional>
    using std::optional;
#else
    #include <experimental/optional>
    using std::experimental::optional;
#endif
```

---

## 十五、IO流

### 15.1 标准输出

```cpp
#include <iostream>

// std::cout
std::cout << "Hello, World!" << std::endl;

// std::cerr（标准错误输出）
std::cerr << "Error message" << std::endl;

// std::clog（标准日志输出）
std::clog << "Log message" << std::endl;

// 格式化输出
int value = 42;
std::cout << "Value: " << value << std::endl;
```

### 15.2 标准输入

```cpp
#include <iostream>

// std::cin
int value;
std::cin >> value;

// 多个输入
int a, b;
std::cin >> a >> b;

// 读取一行
std::string line;
std::getline(std::cin, line);

// 读取单词
std::string word;
std::cin >> word;
```

### 15.3 文件流

```cpp
#include <fstream>

// 写文件
std::ofstream outfile("output.txt");
if (outfile.is_open()) {
    outfile << "Hello, File!" << std::endl;
    outfile.close();
}

// 读文件
std::ifstream infile("input.txt");
if (infile.is_open()) {
    std::string line;
    while (std::getline(infile, line)) {
        std::cout << line << std::endl;
    }
    infile.close();
}

// 读写文件
std::fstream file("data.txt", std::ios::in | std::ios::out);
if (file.is_open()) {
    // 读写操作
    file.close();
}
```

### 15.4 字符串流

```cpp
#include <sstream>

// 输出字符串流
std::ostringstream oss;
oss << "Hello, " << "World!";
std::string str = oss.str();

// 输入字符串流
std::string input = "42 3.14";
std::istringstream iss(input);
int i;
double d;
iss >> i >> d;

// 双向字符串流
std::stringstream ss;
ss << 42 << " " << 3.14;
int x;
double y;
ss >> x >> y;
```

### 15.5 格式化输出

```cpp
#include <iomanip>

// 设置精度
std::cout << std::setprecision(5) << 3.14159 << std::endl;

// 设置宽度
std::cout << std::setw(10) << "Hello" << std::endl;

// 填充字符
std::cout << std::setfill('*') << std::setw(10) << 42 << std::endl;

// 对齐
std::cout << std::left << std::setw(10) << "Left" << std::endl;
std::cout << std::right << std::setw(10) << "Right" << std::endl;

// 科学计数法
std::cout << std::scientific << 3.14159 << std::endl;

// 固定小数点
std::cout << std::fixed << std::setprecision(2) << 3.14159 << std::endl;

// 十六进制
std::cout << std::hex << 255 << std::endl;

// 八进制
std::cout << std::oct << 255 << std::endl;

// 布尔值
std::cout << std::boolalpha << true << std::endl;

// std::format（C++20）
#include <format>
std::string formatted = std::format("Hello {}, value: {}", "World", 42);
std::cout << formatted << std::endl;
```

---

## 总结

本文档涵盖了C++的核心语法规则，包括：

1. **基础语法**：程序结构、注释、标识符
2. **变量与数据类型**：基本类型、类型修饰符、类型推导、类型转换
3. **运算符**：算术、关系、逻辑、位运算、赋值等
4. **控制流**：if、switch、循环、跳转语句
5. **函数**：定义、声明、参数传递、重载、lambda等
6. **数组与字符串**：C风格和现代C++数组、字符串操作
7. **指针与引用**：指针、引用、右值引用、智能指针
8. **结构体与联合体**：结构体、联合体、枚举
9. **类与对象**：类定义、构造函数、析构函数、成员函数、运算符重载
10. **继承与多态**：继承、虚函数、多态、虚继承
11. **模板**：函数模板、类模板、特化、变参模板、Concepts
12. **异常处理**：基本异常处理、自定义异常、noexcept
13. **命名空间**：定义、using声明、别名
14. **预处理器**：宏定义、条件编译、编译时断言
15. **IO流**：标准IO、文件流、字符串流、格式化输出

掌握这些语法规则将帮助您深入理解C++语言，编写高质量的代码！