# STL高级技术（三）：迭代器与算法

## 目录
1. [迭代器深入剖析](#一迭代器深入剖析)
2. [算法设计与实现](#二算法设计与实现)
3. [函数对象与Lambda](#三函数对象与lambda)
4. [适配器与代理](#四适配器与代理)
5. [性能优化技术](#五性能优化技术)

---

## 一、迭代器深入剖析

### 1.1 迭代器的分类与特性

#### 1.1.1 迭代器分类层次

```cpp
// 迭代器标签层次结构
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};

// 迭代器特性萃取
template<typename Iterator>
struct iterator_traits {
    using iterator_category = typename Iterator::iterator_category;
    using value_type = typename Iterator::value_type;
    using difference_type = typename Iterator::difference_type;
    using pointer = typename Iterator::pointer;
    using reference = typename Iterator::reference;
};

// 指针的特化
template<typename T>
struct iterator_traits<T*> {
    using iterator_category = random_access_iterator_tag;
    using value_type = T;
    using difference_type = ptrdiff_t;
    using pointer = T*;
    using reference = T&;
};
```

#### 1.1.2 输入迭代器实现

```cpp
// 输入迭代器：只能向前移动，单遍扫描
template<typename T>
class InputIterator {
public:
    using iterator_category = input_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = const T*;
    using reference = const T&;
    
    InputIterator() : ptr_(nullptr) {}
    
    explicit InputIterator(const T* ptr) : ptr_(ptr) {}
    
    // 解引用
    reference operator*() const {
        return *ptr_;
    }
    
    pointer operator->() const {
        return ptr_;
    }
    
    // 前置递增
    InputIterator& operator++() {
        ++ptr_;
        return *this;
    }
    
    // 后置递增
    InputIterator operator++(int) {
        InputIterator temp = *this;
        ++ptr_;
        return temp;
    }
    
    // 相等比较
    bool operator==(const InputIterator& other) const {
        return ptr_ == other.ptr_;
    }
    
    bool operator!=(const InputIterator& other) const {
        return ptr_ != other.ptr_;
    }
    
private:
    const T* ptr_;
};
```

#### 1.1.3 前向迭代器实现

```cpp
// 前向迭代器：可以多次遍历
template<typename T>
class ForwardIterator {
public:
    using iterator_category = forward_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using reference = T&;
    
    ForwardIterator() : ptr_(nullptr) {}
    
    explicit ForwardIterator(T* ptr) : ptr_(ptr) {}
    
    reference operator*() const {
        return *ptr_;
    }
    
    pointer operator->() const {
        return ptr_;
    }
    
    ForwardIterator& operator++() {
        ++ptr_;
        return *this;
    }
    
    ForwardIterator operator++(int) {
        ForwardIterator temp = *this;
        ++ptr_;
        return temp;
    }
    
    bool operator==(const ForwardIterator& other) const {
        return ptr_ == other.ptr_;
    }
    
    bool operator!=(const ForwardIterator& other) const {
        return ptr_ != other.ptr_;
    }
    
private:
    T* ptr_;
};
```

#### 1.1.4 双向迭代器实现

```cpp
// 双向迭代器：可以前后移动
template<typename T>
class BidirectionalIterator {
public:
    using iterator_category = bidirectional_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using reference = T&;
    
    BidirectionalIterator() : ptr_(nullptr) {}
    
    explicit BidirectionalIterator(T* ptr) : ptr_(ptr) {}
    
    reference operator*() const {
        return *ptr_;
    }
    
    pointer operator->() const {
        return ptr_;
    }
    
    BidirectionalIterator& operator++() {
        ++ptr_;
        return *this;
    }
    
    BidirectionalIterator operator++(int) {
        BidirectionalIterator temp = *this;
        ++ptr_;
        return temp;
    }
    
    BidirectionalIterator& operator--() {
        --ptr_;
        return *this;
    }
    
    BidirectionalIterator operator--(int) {
        BidirectionalIterator temp = *this;
        --ptr_;
        return temp;
    }
    
    bool operator==(const BidirectionalIterator& other) const {
        return ptr_ == other.ptr_;
    }
    
    bool operator!=(const BidirectionalIterator& other) const {
        return ptr_ != other.ptr_;
    }
    
private:
    T* ptr_;
};
```

#### 1.1.5 随机访问迭代器实现

```cpp
// 随机访问迭代器：支持任意位置访问和算术运算
template<typename T>
class RandomAccessIterator {
public:
    using iterator_category = random_access_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using reference = T&;
    
    RandomAccessIterator() : ptr_(nullptr) {}
    
    explicit RandomAccessIterator(T* ptr) : ptr_(ptr) {}
    
    reference operator*() const {
        return *ptr_;
    }
    
    pointer operator->() const {
        return ptr_;
    }
    
    reference operator[](difference_type n) const {
        return ptr_[n];
    }
    
    RandomAccessIterator& operator++() {
        ++ptr_;
        return *this;
    }
    
    RandomAccessIterator operator++(int) {
        RandomAccessIterator temp = *this;
        ++ptr_;
        return temp;
    }
    
    RandomAccessIterator& operator--() {
        --ptr_;
        return *this;
    }
    
    RandomAccessIterator operator--(int) {
        RandomAccessIterator temp = *this;
        --ptr_;
        return temp;
    }
    
    RandomAccessIterator& operator+=(difference_type n) {
        ptr_ += n;
        return *this;
    }
    
    RandomAccessIterator operator+(difference_type n) const {
        return RandomAccessIterator(ptr_ + n);
    }
    
    RandomAccessIterator& operator-=(difference_type n) {
        ptr_ -= n;
        return *this;
    }
    
    RandomAccessIterator operator-(difference_type n) const {
        return RandomAccessIterator(ptr_ - n);
    }
    
    difference_type operator-(const RandomAccessIterator& other) const {
        return ptr_ - other.ptr_;
    }
    
    bool operator==(const RandomAccessIterator& other) const {
        return ptr_ == other.ptr_;
    }
    
    bool operator!=(const RandomAccessIterator& other) const {
        return ptr_ != other.ptr_;
    }
    
    bool operator<(const RandomAccessIterator& other) const {
        return ptr_ < other.ptr_;
    }
    
    bool operator>(const RandomAccessIterator& other) const {
        return ptr_ > other.ptr_;
    }
    
    bool operator<=(const RandomAccessIterator& other) const {
        return ptr_ <= other.ptr_;
    }
    
    bool operator>=(const RandomAccessIterator& other) const {
        return ptr_ >= other.ptr_;
    }
    
private:
    T* ptr_;
};

// 非成员运算符
template<typename T>
RandomAccessIterator<T> operator+(
    typename RandomAccessIterator<T>::difference_type n,
    const RandomAccessIterator<T>& it) {
    return it + n;
}
```

### 1.2 迭代器适配器

#### 1.2.1 反向迭代器

```cpp
// 反向迭代器适配器
template<typename Iterator>
class ReverseIterator {
public:
    using iterator_type = Iterator;
    using iterator_category = typename std::iterator_traits<Iterator>::iterator_category;
    using value_type = typename std::iterator_traits<Iterator>::value_type;
    using difference_type = typename std::iterator_traits<Iterator>::difference_type;
    using pointer = typename std::iterator_traits<Iterator>::pointer;
    using reference = typename std::iterator_traits<Iterator>::reference;
    
    ReverseIterator() : current_() {}
    
    explicit ReverseIterator(Iterator it) : current_(it) {}
    
    Iterator base() const {
        return current_;
    }
    
    reference operator*() const {
        Iterator temp = current_;
        return *--temp;
    }
    
    pointer operator->() const {
        return &(operator*());
    }
    
    ReverseIterator& operator++() {
        --current_;
        return *this;
    }
    
    ReverseIterator operator++(int) {
        ReverseIterator temp = *this;
        --current_;
        return temp;
    }
    
    ReverseIterator& operator--() {
        ++current_;
        return *this;
    }
    
    ReverseIterator operator--(int) {
        ReverseIterator temp = *this;
        ++current_;
        return temp;
    }
    
    ReverseIterator& operator+=(difference_type n) {
        current_ -= n;
        return *this;
    }
    
    ReverseIterator operator+(difference_type n) const {
        return ReverseIterator(current_ - n);
    }
    
    ReverseIterator& operator-=(difference_type n) {
        current_ += n;
        return *this;
    }
    
    ReverseIterator operator-(difference_type n) const {
        return ReverseIterator(current_ + n);
    }
    
    difference_type operator-(const ReverseIterator& other) const {
        return other.current_ - current_;
    }
    
    reference operator[](difference_type n) const {
        return *(*this + n);
    }
    
    bool operator==(const ReverseIterator& other) const {
        return current_ == other.current_;
    }
    
    bool operator!=(const ReverseIterator& other) const {
        return current_ != other.current_;
    }
    
    bool operator<(const ReverseIterator& other) const {
        return current_ > other.current_;
    }
    
    bool operator>(const ReverseIterator& other) const {
        return current_ < other.current_;
    }
    
    bool operator<=(const ReverseIterator& other) const {
        return current_ >= other.current_;
    }
    
    bool operator>=(const ReverseIterator& other) const {
        return current_ <= other.current_;
    }
    
private:
    Iterator current_;
};

// 使用
std::vector<int> vec = {1, 2, 3, 4, 5};
ReverseIterator<std::vector<int>::iterator> rbegin(vec.end());
ReverseIterator<std::vector<int>::iterator> rend(vec.begin());

for (auto it = rbegin; it != rend; ++it) {
    std::cout << *it << " ";  // 5 4 3 2 1
}
```

#### 1.2.2 插入迭代器

```cpp
// 插入迭代器基类
template<typename Container>
class InsertIterator {
public:
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = void;
    using pointer = void;
    using reference = void;
    using container_type = Container;
    
    explicit InsertIterator(Container& c, typename Container::iterator it)
        : container_(std::addressof(c)), iter_(it) {}
    
    InsertIterator& operator=(const typename Container::value_type& value) {
        iter_ = container_->insert(iter_, value);
        ++iter_;
        return *this;
    }
    
    InsertIterator& operator=(typename Container::value_type&& value) {
        iter_ = container_->insert(iter_, std::move(value));
        ++iter_;
        return *this;
    }
    
    InsertIterator& operator*() {
        return *this;
    }
    
    InsertIterator& operator++() {
        return *this;
    }
    
    InsertIterator& operator++(int) {
        return *this;
    }
    
protected:
    Container* container_;
    typename Container::iterator iter_;
};

// 后插迭代器
template<typename Container>
class BackInsertIterator : public InsertIterator<Container> {
public:
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = void;
    using pointer = void;
    using reference = void;
    using container_type = Container;
    
    explicit BackInsertIterator(Container& c)
        : container_(std::addressof(c)) {}
    
    BackInsertIterator& operator=(const typename Container::value_type& value) {
        container_->push_back(value);
        return *this;
    }
    
    BackInsertIterator& operator=(typename Container::value_type&& value) {
        container_->push_back(std::move(value));
        return *this;
    }
    
    BackInsertIterator& operator*() {
        return *this;
    }
    
    BackInsertIterator& operator++() {
        return *this;
    }
    
    BackInsertIterator& operator++(int) {
        return *this;
    }
    
protected:
    Container* container_;
};

// 前插迭代器
template<typename Container>
class FrontInsertIterator : public InsertIterator<Container> {
public:
    using iterator_category = std::output_iterator_tag;
    using value_type = void;
    using difference_type = void;
    using pointer = void;
    using reference = void;
    using container_type = Container;
    
    explicit FrontInsertIterator(Container& c)
        : container_(std::addressof(c)) {}
    
    FrontInsertIterator& operator=(const typename Container::value_type& value) {
        container_->push_front(value);
        return *this;
    }
    
    FrontInsertIterator& operator=(typename Container::value_type&& value) {
        container_->push_front(std::move(value));
        return *this;
    }
    
    FrontInsertIterator& operator*() {
        return *this;
    }
    
    FrontInsertIterator& operator++() {
        return *this;
    }
    
    FrontInsertIterator& operator++(int) {
        return *this;
    }
    
protected:
    Container* container_;
};

// 工厂函数
template<typename Container>
BackInsertIterator<Container> back_inserter(Container& c) {
    return BackInsertIterator<Container>(c);
}

template<typename Container>
FrontInsertIterator<Container> front_inserter(Container& c) {
    return FrontInsertIterator<Container>(c);
}

// 使用
std::vector<int> source = {1, 2, 3, 4, 5};
std::vector<int> dest;
std::list<int> dest2;

// 复制到vector后面
std::copy(source.begin(), source.end(), back_inserter(dest));

// 复制到list前面
std::copy(source.begin(), source.end(), front_inserter(dest2));
```

### 1.3 迭代器辅助函数

```cpp
// advance：将迭代器前进n步
template<typename InputIterator, typename Distance>
void advance(InputIterator& it, Distance n) {
    using category = typename std::iterator_traits<InputIterator>::iterator_category;
    
    if constexpr (std::is_same_v<category, std::random_access_iterator_tag>) {
        it += n;  // O(1)
    } else {
        if (n > 0) {
            for (; n > 0; --n, ++it) {}
        } else if constexpr (std::is_same_v<category, std::bidirectional_iterator_tag>) {
            for (; n < 0; ++n, --it) {}
        }
        // O(n)
    }
}

// distance：计算两个迭代器之间的距离
template<typename InputIterator>
typename std::iterator_traits<InputIterator>::difference_type
distance(InputIterator first, InputIterator last) {
    using category = typename std::iterator_traits<InputIterator>::iterator_category;
    
    if constexpr (std::is_same_v<category, std::random_access_iterator_tag>) {
        return last - first;  // O(1)
    } else {
        typename std::iterator_traits<InputIterator>::difference_type dist = 0;
        for (; first != last; ++first, ++dist) {}
        return dist;  // O(n)
    }
}

// next：获取下一个迭代器
template<typename InputIterator>
InputIterator next(InputIterator it,
                  typename std::iterator_traits<InputIterator>::difference_type n = 1) {
    advance(it, n);
    return it;
}

// prev：获取前一个迭代器
template<typename BidirectionalIterator>
BidirectionalIterator prev(BidirectionalIterator it,
                          typename std::iterator_traits<BidirectionalIterator>::difference_type n = 1) {
    advance(it, -n);
    return it;
}
```

---

## 二、算法设计与实现

### 2.1 非修改序列算法

#### 2.1.1 find算法

```cpp
// find：查找第一个满足条件的元素
template<typename InputIterator, typename T>
InputIterator find(InputIterator first, InputIterator last, const T& value) {
    for (; first != last; ++first) {
        if (*first == value) {
            return first;
        }
    }
    return last;
}

// find_if：查找第一个满足谓词的元素
template<typename InputIterator, typename Predicate>
InputIterator find_if(InputIterator first, InputIterator last, Predicate pred) {
    for (; first != last; ++first) {
        if (pred(*first)) {
            return first;
        }
    }
    return last;
}

// find_if_not：查找第一个不满足谓词的元素
template<typename InputIterator, typename Predicate>
InputIterator find_if_not(InputIterator first, InputIterator last, Predicate pred) {
    for (; first != last; ++first) {
        if (!pred(*first)) {
            return first;
        }
    }
    return last;
}

// 使用
std::vector<int> vec = {1, 2, 3, 4, 5};
auto it1 = find(vec.begin(), vec.end(), 3);
auto it2 = find_if(vec.begin(), vec.end(), [](int x) { return x > 3; });
auto it3 = find_if_not(vec.begin(), vec.end(), [](int x) { return x < 3; });
```

#### 2.1.2 count算法

```cpp
// count：统计等于value的元素个数
template<typename InputIterator, typename T>
typename std::iterator_traits<InputIterator>::difference_type
count(InputIterator first, InputIterator last, const T& value) {
    typename std::iterator_traits<InputIterator>::difference_type count = 0;
    for (; first != last; ++first) {
        if (*first == value) {
            ++count;
        }
    }
    return count;
}

// count_if：统计满足谓词的元素个数
template<typename InputIterator, typename Predicate>
typename std::iterator_traits<InputIterator>::difference_type
count_if(InputIterator first, InputIterator last, Predicate pred) {
    typename std::iterator_traits<InputIterator>::difference_type count = 0;
    for (; first != last; ++first) {
        if (pred(*first)) {
            ++count;
        }
    }
    return count;
}

// 使用
std::vector<int> vec = {1, 2, 2, 3, 2, 4, 5};
auto count1 = count(vec.begin(), vec.end(), 2);  // 3
auto count2 = count_if(vec.begin(), vec.end(), [](int x) { return x > 2; });  // 3
```

#### 2.1.3 all_of/any_of/none_of

```cpp
// all_of：是否所有元素都满足谓词
template<typename InputIterator, typename Predicate>
bool all_of(InputIterator first, InputIterator last, Predicate pred) {
    for (; first != last; ++first) {
        if (!pred(*first)) {
            return false;
        }
    }
    return true;
}

// any_of：是否至少有一个元素满足谓词
template<typename InputIterator, typename Predicate>
bool any_of(InputIterator first, InputIterator last, Predicate pred) {
    for (; first != last; ++first) {
        if (pred(*first)) {
            return true;
        }
    }
    return false;
}

// none_of：是否所有元素都不满足谓词
template<typename InputIterator, typename Predicate>
bool none_of(InputIterator first, InputIterator last, Predicate pred) {
    for (; first != last; ++first) {
        if (pred(*first)) {
            return false;
        }
    }
    return true;
}

// 使用
std::vector<int> vec = {2, 4, 6, 8, 10};
bool all_even = all_of(vec.begin(), vec.end(), [](int x) { return x % 2 == 0; });
bool has_odd = any_of(vec.begin(), vec.end(), [](int x) { return x % 2 != 0; });
bool none_negative = none_of(vec.begin(), vec.end(), [](int x) { return x < 0; });
```

### 2.2 修改序列算法

#### 2.2.1 copy算法

```cpp
// copy：复制元素
template<typename InputIterator, typename OutputIterator>
OutputIterator copy(InputIterator first, InputIterator last, OutputIterator d_first) {
    for (; first != last; ++first, ++d_first) {
        *d_first = *first;
    }
    return d_first;
}

// copy_if：复制满足条件的元素
template<typename InputIterator, typename OutputIterator, typename Predicate>
OutputIterator copy_if(InputIterator first, InputIterator last, 
                      OutputIterator d_first, Predicate pred) {
    for (; first != last; ++first) {
        if (pred(*first)) {
            *d_first++ = *first;
        }
    }
    return d_first;
}

// copy_n：复制n个元素
template<typename InputIterator, typename Size, typename OutputIterator>
OutputIterator copy_n(InputIterator first, Size count, OutputIterator d_first) {
    for (Size i = 0; i < count; ++i) {
        *d_first++ = *first++;
    }
    return d_first;
}

// move：移动元素
template<typename InputIterator, typename OutputIterator>
OutputIterator move(InputIterator first, InputIterator last, OutputIterator d_first) {
    for (; first != last; ++first, ++d_first) {
        *d_first = std::move(*first);
    }
    return d_first;
}

// 使用
std::vector<int> source = {1, 2, 3, 4, 5};
std::vector<int> dest(source.size());
std::vector<int> dest_filtered;
std::vector<std::string> strings = {"hello", "world"};

copy(source.begin(), source.end(), dest.begin());
copy_if(source.begin(), source.end(), back_inserter(dest_filtered), [](int x) { return x > 2; });
std::vector<std::string> moved_strings(strings.size());
move(strings.begin(), strings.end(), moved_strings.begin());
```

#### 2.2.2 transform算法

```cpp
// transform（一元）：转换元素
template<typename InputIterator, typename OutputIterator, typename UnaryOperation>
OutputIterator transform(InputIterator first, InputIterator last,
                        OutputIterator d_first, UnaryOperation unary_op) {
    for (; first != last; ++first, ++d_first) {
        *d_first = unary_op(*first);
    }
    return d_first;
}

// transform（二元）：转换两个序列
template<typename InputIterator1, typename InputIterator2, 
         typename OutputIterator, typename BinaryOperation>
OutputIterator transform(InputIterator1 first1, InputIterator1 last1,
                        InputIterator2 first2, OutputIterator d_first,
                        BinaryOperation binary_op) {
    for (; first1 != last1; ++first1, ++first2, ++d_first) {
        *d_first = binary_op(*first1, *first2);
    }
    return d_first;
}

// 使用
std::vector<int> vec = {1, 2, 3, 4, 5};
std::vector<int> squared;
std::vector<int> doubled;

transform(vec.begin(), vec.end(), back_inserter(squared), [](int x) { return x * x; });
transform(vec.begin(), vec.end(), back_inserter(doubled), [](int x) { return x * 2; });

std::vector<int> vec1 = {1, 2, 3};
std::vector<int> vec2 = {4, 5, 6};
std::vector<int> sum;

transform(vec1.begin(), vec1.end(), vec2.begin(), back_inserter(sum),
          [](int a, int b) { return a + b; });
```

#### 2.2.3 remove算法

```cpp
// remove：移除等于value的元素（不真正删除）
template<typename ForwardIterator, typename T>
ForwardIterator remove(ForwardIterator first, ForwardIterator last, const T& value) {
    first = std::find(first, last, value);
    if (first != last) {
        for (ForwardIterator i = first; ++i != last; ) {
            if (!(*i == value)) {
                *first++ = std::move(*i);
            }
        }
    }
    return first;
}

// remove_if：移除满足条件的元素
template<typename ForwardIterator, typename Predicate>
ForwardIterator remove_if(ForwardIterator first, ForwardIterator last, Predicate pred) {
    first = std::find_if(first, last, pred);
    if (first != last) {
        for (ForwardIterator i = first; ++i != last; ) {
            if (!pred(*i)) {
                *first++ = std::move(*i);
            }
        }
    }
    return first;
}

// erase-remove惯用法
template<typename Container, typename T>
void erase_remove(Container& c, const T& value) {
    c.erase(std::remove(c.begin(), c.end(), value), c.end());
}

template<typename Container, typename Predicate>
void erase_remove_if(Container& c, Predicate pred) {
    c.erase(std::remove_if(c.begin(), c.end(), pred), c.end());
}

// 使用
std::vector<int> vec = {1, 2, 3, 2, 4, 2, 5};
erase_remove(vec, 2);  // {1, 3, 4, 5}

erase_remove_if(vec, [](int x) { return x % 2 != 0; });  // {4}
```

### 2.3 排序算法

#### 2.3.1 sort算法

```cpp
// 快速排序实现
template<typename RandomAccessIterator, typename Compare>
void quick_sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    if (last - first <= 1) return;
    
    auto pivot = *(first + (last - first) / 2);
    auto left = first;
    auto right = last - 1;
    
    while (left <= right) {
        while (comp(*left, pivot)) ++left;
        while (comp(pivot, *right)) --right;
        if (left <= right) {
            std::iter_swap(left++, right--);
        }
    }
    
    quick_sort(first, right + 1, comp);
    quick_sort(left, last, comp);
}

// 插入排序实现
template<typename RandomAccessIterator, typename Compare>
void insertion_sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    for (auto i = first + 1; i != last; ++i) {
        auto key = std::move(*i);
        auto j = i;
        
        while (j != first && comp(key, *(j - 1))) {
            *j = std::move(*(j - 1));
            --j;
        }
        
        *j = std::move(key);
    }
}

// 混合排序：对小范围使用插入排序
template<typename RandomAccessIterator, typename Compare>
void hybrid_sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    constexpr int threshold = 16;
    
    if (last - first <= threshold) {
        insertion_sort(first, last, comp);
    } else {
        quick_sort(first, last, comp);
    }
}

// sort：排序算法
template<typename RandomAccessIterator>
void sort(RandomAccessIterator first, RandomAccessIterator last) {
    sort(first, last, std::less<>());
}

template<typename RandomAccessIterator, typename Compare>
void sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    hybrid_sort(first, last, comp);
}

// 使用
std::vector<int> vec = {5, 2, 8, 1, 9, 3, 7, 4, 6};
sort(vec.begin(), vec.end());  // 升序
sort(vec.begin(), vec.end(), std::greater<>());  // 降序
```

#### 2.3.2 稳定排序

```cpp
// 归并排序实现
template<typename RandomAccessIterator, typename Compare>
void merge_sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    auto size = last - first;
    if (size <= 1) return;
    
    auto mid = first + size / 2;
    merge_sort(first, mid, comp);
    merge_sort(mid, last, comp);
    
    std::vector<typename std::iterator_traits<RandomAccessIterator>::value_type> temp(size);
    std::merge(first, mid, mid, last, temp.begin(), comp);
    std::copy(temp.begin(), temp.end(), first);
}

// stable_sort：稳定排序
template<typename RandomAccessIterator>
void stable_sort(RandomAccessIterator first, RandomAccessIterator last) {
    stable_sort(first, last, std::less<>());
}

template<typename RandomAccessIterator, typename Compare>
void stable_sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    merge_sort(first, last, comp);
}

// 使用
struct Person {
    std::string name;
    int age;
    
    bool operator<(const Person& other) const {
        return age < other.age;
    }
};

std::vector<Person> people = {
    {"Alice", 25}, {"Bob", 30}, {"Charlie", 25}
};

stable_sort(people.begin(), people.end());
// Alice(25), Charlie(25), Bob(30) - 保持年龄相同者的原始顺序
```

#### 2.3.3 堆操作

```cpp
// heap相关操作
template<typename RandomAccessIterator, typename Compare>
void make_heap(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    auto size = last - first;
    for (auto i = size / 2 - 1; i >= 0; --i) {
        heapify_down(first + i, first, last, comp);
    }
}

template<typename RandomAccessIterator, typename Compare>
void heapify_down(RandomAccessIterator first, RandomAccessIterator first_root,
                 RandomAccessIterator last, Compare comp) {
    auto size = last - first;
    auto root = first_root - first;
    
    while (true) {
        auto largest = root;
        auto left = 2 * root + 1;
        auto right = 2 * root + 2;
        
        if (left < size && comp(first[largest], first[left])) {
            largest = left;
        }
        
        if (right < size && comp(first[largest], first[right])) {
            largest = right;
        }
        
        if (largest == root) break;
        
        std::iter_swap(first + root, first + largest);
        root = largest;
    }
}

template<typename RandomAccessIterator, typename Compare>
void push_heap(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    if (last - first <= 1) return;
    
    auto size = last - first;
    auto child = size - 1;
    auto parent = (child - 1) / 2;
    
    while (child > 0 && comp(first[parent], first[child])) {
        std::iter_swap(first + parent, first + child);
        child = parent;
        parent = (child - 1) / 2;
    }
}

template<typename RandomAccessIterator, typename Compare>
void pop_heap(RandomAccessIterator first, RandomAccessIterator last, Compare comp) {
    if (last - first <= 1) return;
    
    std::iter_swap(first, last - 1);
    heapify_down(first, first, last - 1, comp);
}

// 使用
std::vector<int> vec = {3, 1, 4, 1, 5, 9, 2, 6};
make_heap(vec.begin(), vec.end());
// 最大堆：[9, 6, 4, 1, 5, 3, 2, 1]

vec.push_back(8);
push_heap(vec.begin(), vec.end());

pop_heap(vec.begin(), vec.end());
vec.pop_back();
```

### 2.4 查找算法

#### 2.4.1 binary_search

```cpp
// lower_bound：第一个不小于value的位置
template<typename ForwardIterator, typename T, typename Compare>
ForwardIterator lower_bound(ForwardIterator first, ForwardIterator last, 
                           const T& value, Compare comp) {
    auto count = last - first;
    
    while (count > 0) {
        auto step = count / 2;
        auto it = first + step;
        
        if (comp(*it, value)) {
            first = ++it;
            count -= step + 1;
        } else {
            count = step;
        }
    }
    
    return first;
}

// upper_bound：第一个大于value的位置
template<typename ForwardIterator, typename T, typename Compare>
ForwardIterator upper_bound(ForwardIterator first, ForwardIterator last,
                           const T& value, Compare comp) {
    auto count = last - first;
    
    while (count > 0) {
        auto step = count / 2;
        auto it = first + step;
        
        if (!comp(value, *it)) {
            first = ++it;
            count -= step + 1;
        } else {
            count = step;
        }
    }
    
    return first;
}

// binary_search：二分查找
template<typename ForwardIterator, typename T, typename Compare>
bool binary_search(ForwardIterator first, ForwardIterator last,
                  const T& value, Compare comp) {
    first = lower_bound(first, last, value, comp);
    return (!(first == last) && !(comp(value, *first)));
}

// 使用
std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
auto lower = lower_bound(vec.begin(), vec.end(), 5);  // 指向5
auto upper = upper_bound(vec.begin(), vec.end(), 5);  // 指向6
bool found = binary_search(vec.begin(), vec.end(), 5);  // true
```

---

## 三、函数对象与Lambda

### 3.1 函数对象实现

#### 3.1.1 算术函数对象

```cpp
// 算术运算
template<typename T = void>
struct plus {
    constexpr T operator()(const T& lhs, const T& rhs) const {
        return lhs + rhs;
    }
};

template<>
struct plus<void> {
    template<typename T, typename U>
    constexpr auto operator()(T&& lhs, U&& rhs) const
        -> decltype(std::forward<T>(lhs) + std::forward<U>(rhs)) {
        return std::forward<T>(lhs) + std::forward<U>(rhs);
    }
};

template<typename T = void>
struct minus {
    constexpr T operator()(const T& lhs, const T& rhs) const {
        return lhs - rhs;
    }
};

template<typename T = void>
struct multiplies {
    constexpr T operator()(const T& lhs, const T& rhs) const {
        return lhs * rhs;
    }
};

template<typename T = void>
struct divides {
    constexpr T operator()(const T& lhs, const T& rhs) const {
        return lhs / rhs;
    }
};

template<typename T = void>
struct modulus {
    constexpr T operator()(const T& lhs, const T& rhs) const {
        return lhs % rhs;
    }
};

template<typename T = void>
struct negate {
    constexpr T operator()(const T& arg) const {
        return -arg;
    }
};

// 使用
plus<> add;
minus<> sub;
multiplies<> mul;

int result1 = add(10, 20);      // 30
int result2 = sub(20, 10);      // 10
int result3 = mul(5, 6);        // 30
```

#### 3.1.2 比较函数对象

```cpp
// 比较运算
template<typename T = void>
struct equal_to {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs == rhs;
    }
};

template<>
struct equal_to<void> {
    template<typename T, typename U>
    constexpr auto operator()(T&& lhs, U&& rhs) const
        -> decltype(std::forward<T>(lhs) == std::forward<U>(rhs)) {
        return std::forward<T>(lhs) == std::forward<U>(rhs);
    }
};

template<typename T = void>
struct not_equal_to {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs != rhs;
    }
};

template<typename T = void>
struct greater {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs > rhs;
    }
};

template<typename T = void>
struct less {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs < rhs;
    }
};

template<typename T = void>
struct greater_equal {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs >= rhs;
    }
};

template<typename T = void>
struct less_equal {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs <= rhs;
    }
};

// 使用
less<> lt;
greater<> gt;

bool result1 = lt(10, 20);   // true
bool result2 = gt(10, 20);   // false
```

#### 3.1.3 逻辑函数对象

```cpp
// 逻辑运算
template<typename T = void>
struct logical_and {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs && rhs;
    }
};

template<typename T = void>
struct logical_or {
    constexpr bool operator()(const T& lhs, const T& rhs) const {
        return lhs || rhs;
    }
};

template<typename T = void>
struct logical_not {
    constexpr bool operator()(const T& arg) const {
        return !arg;
    }
};

// 使用
logical_and<> land;
logical_or<> lor;
logical_not<> lnot;

bool result1 = land(true, false);  // false
bool result2 = lor(true, false);   // true
bool result3 = lnot(true);         // false
```

### 3.2 Lambda表达式深入

#### 3.2.1 Lambda实现原理

```cpp
// Lambda的编译器转换
// [capture](params) -> return_type { body }

// 示例Lambda
auto lambda = [](int x, int y) { return x + y; };

// 编译器转换后的等价类
class CompilerGeneratedLambda {
public:
    CompilerGeneratedLambda() = default;
    
    int operator()(int x, int y) const {
        return x + y;
    }
};

auto lambda2 = CompilerGeneratedLambda();
```

#### 3.2.2 捕获机制实现

```cpp
// 按值捕获
int a = 10, b = 20;
auto lambda1 = [a, b](int x) { return x + a + b; };

// 等价类
class LambdaWithValueCapture {
    int a_;
    int b_;
public:
    LambdaWithValueCapture(int a, int b) : a_(a), b_(b) {}
    
    int operator()(int x) const {
        return x + a_ + b_;
    }
};

// 按引用捕获
auto lambda2 = [&a, &b](int x) { return x + a + b; };

// 等价类
class LambdaWithReferenceCapture {
    int& a_;
    int& b_;
public:
    LambdaWithReferenceCapture(int& a, int& b) : a_(a), b_(b) {}
    
    int operator()(int x) const {
        return x + a_ + b_;
    }
};

// 按值捕获所有（=）
auto lambda3 = [=](int x) { return x + a + b; };

// 等价类
class LambdaWithCaptureAllByValue {
    int a_;
    int b_;
public:
    LambdaWithCaptureAllByValue(int a, int b) : a_(a), b_(b) {}
    
    int operator()(int x) const {
        return x + a_ + b_;
    }
};

// 按引用捕获所有（&）
auto lambda4 = [&](int x) { return x + a + b; };

// 等价类
class LambdaWithCaptureAllByReference {
    int& a_;
    int& b_;
public:
    LambdaWithCaptureAllByReference(int& a, int& b) : a_(a), b_(b) {}
    
    int operator()(int x) const {
        return x + a_ + b_;
    }
};
```

#### 3.2.3 mutable Lambda

```cpp
// mutable Lambda：可以修改捕获的副本
int a = 10;
auto lambda = [a]() mutable {
    a = 100;  // 修改副本
    return a;
};

// 等价类
class MutableLambda {
    int a_;
public:
    MutableLambda(int a) : a_(a) {}
    
    int operator()() {
        a_ = 100;
        return a_;
    }
};

// 使用
int result1 = lambda();  // 100
int result2 = lambda();  // 100（每次都是新的副本）
std::cout << a << std::endl;  // 10（原始值不变）
```

### 3.3 高阶函数

```cpp
// 函数组合
template<typename F, typename G>
auto compose(F&& f, G&& g) {
    return [f = std::forward<F>(f), g = std::forward<G>(g)](auto&&... args) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

// 使用
auto square = [](int x) { return x * x; };
auto increment = [](int x) { return x + 1; };

auto square_then_increment = compose(increment, square);
auto increment_then_square = compose(square, increment);

int result1 = square_then_increment(5);  // 26 = 5*5 + 1
int result2 = increment_then_square(5);  // 36 = (5+1)*6

// 函数管道
template<typename F>
auto pipeline(F&& f) {
    return [f = std::forward<F>(f)](auto&& x) {
        return f(std::forward<decltype(x)>(x));
    };
}

template<typename F, typename... Fs>
auto pipeline(F&& f, Fs&&... fs) {
    return [f = std::forward<F>(f), fs...](auto&& x) mutable {
        auto result = f(std::forward<decltype(x)>(x));
        return pipeline(fs...)(std::move(result));
    };
}

// 使用
auto validate = [](int x) { return x >= 0; };
auto double_value = [](int x) { return x * 2; };
auto to_string = [](int x) { return std::to_string(x); };

auto process = pipeline(validate, double_value, to_string);
auto result = process(5);  // "10"
```

---

## 四、适配器与代理

### 4.1 函数适配器

```cpp
// std::function简化实现
template<typename Signature>
class function;

template<typename R, typename... Args>
class function<R(Args...)> {
public:
    function() = default;
    
    template<typename F>
    function(F&& f) {
        impl_ = new Callable<F>(std::forward<F>(f));
    }
    
    ~function() {
        delete impl_;
    }
    
    R operator()(Args... args) const {
        return impl_->invoke(std::forward<Args>(args)...);
    }
    
private:
    struct CallableBase {
        virtual ~CallableBase() = default;
        virtual R invoke(Args...) = 0;
    };
    
    template<typename F>
    struct Callable : CallableBase {
        F f_;
        
        Callable(F&& f) : f_(std::forward<F>(f)) {}
        
        R invoke(Args... args) override {
            return f_(std::forward<Args>(args)...);
        }
    };
    
    CallableBase* impl_ = nullptr;
};

// 使用
function<int(int, int)> add = [](int a, int b) { return a + b; };
int result = add(10, 20);  // 30
```

### 4.2 std::bind实现

```cpp
// std::bind简化实现
template<typename F, typename... BoundArgs>
class BoundFunction {
public:
    BoundFunction(F&& f, BoundArgs&&... args)
        : f_(std::forward<F>(f)), args_(std::make_tuple(std::forward<BoundArgs>(args)...)) {}
    
    template<typename... CallArgs>
    auto operator()(CallArgs&&... args) const {
        return call_impl(std::index_sequence_for<BoundArgs...>{}, 
                        std::forward<CallArgs>(args)...);
    }
    
private:
    F f_;
    std::tuple<BoundArgs...> args_;
    
    template<typename... CallArgs, std::size_t... Is>
    auto call_impl(std::index_sequence<Is...>, CallArgs&&... args) const {
        return f_(resolve_arg(std::get<Is>(args_), std::forward<CallArgs>(args)...)...);
    }
    
    template<typename T, typename U>
    static auto&& resolve_arg(T&& arg, U&&...) {
        return std::forward<T>(arg);
    }
    
    template<std::size_t I, typename U>
    static auto&& resolve_arg(std::placeholders::_t<I>, U&& arg) {
        return std::forward<U>(arg);
    }
};

// 占位符
namespace placeholders {
    static constexpr std::placeholders::_t<1> _1;
    static constexpr std::placeholders::_t<2> _2;
    static constexpr std::placeholders::_t<3> _3;
}

// bind工厂函数
template<typename F, typename... Args>
auto bind(F&& f, Args&&... args) {
    return BoundFunction<F, Args...>(std::forward<F>(f), 
                                      std::forward<Args>(args)...);
}

// 使用
int multiply(int a, int b) {
    return a * b;
}

auto multiply_by_10 = bind(multiply, 10, placeholders::_1);
int result = multiply_by_10(5);  // 50
```

---

## 五、性能优化技术

### 5.1 循环展开

```cpp
// 循环展开优化
template<typename Iterator, typename Size, typename Function>
void unrolled_for_each(Iterator first, Iterator last, Function func) {
    constexpr size_t unroll_factor = 4;
    
    auto size = std::distance(first, last);
    auto end_unrolled = first + (size / unroll_factor) * unroll_factor;
    
    // 展开循环
    for (auto it = first; it != end_unrolled; it += unroll_factor) {
        func(*it);
        func(*(it + 1));
        func(*(it + 2));
        func(*(it + 3));
    }
    
    // 处理剩余元素
    for (auto it = end_unrolled; it != last; ++it) {
        func(*it);
    }
}

// 使用
std::vector<int> vec(100);
unrolled_for_each(vec.begin(), vec.end(), [](int& x) { x *= 2; });
```

### 5.2 SIMD优化

```cpp
// SIMD优化示例（概念性）
template<typename T>
void simd_add(const T* a, const T* b, T* result, size_t size) {
    constexpr size_t simd_width = 4;  // 假设SIMD宽度为4
    
    // SIMD对齐处理
    size_t simd_size = (size / simd_width) * simd_width;
    
    // SIMD处理
    for (size_t i = 0; i < simd_size; i += simd_width) {
        // SIMD加法（伪代码）
        // __m128 va = _mm_load_ps(&a[i]);
        // __m128 vb = _mm_load_ps(&b[i]);
        // __m128 vr = _mm_add_ps(va, vb);
        // _mm_store_ps(&result[i], vr);
    }
    
    // 处理剩余元素
    for (size_t i = simd_size; i < size; ++i) {
        result[i] = a[i] + b[i];
    }
}
```

### 5.3 缓存友好优化

```cpp
// 缓存友好的矩阵乘法
template<typename T>
void cache_friendly_matrix_multiply(const T* A, const T* B, T* C, 
                                    size_t M, size_t N, size_t K) {
    constexpr size_t block_size = 32;
    
    for (size_t i = 0; i < M; i += block_size) {
        for (size_t j = 0; j < N; j += block_size) {
            for (size_t k = 0; k < K; k += block_size) {
                // 块处理
                size_t i_end = std::min(i + block_size, M);
                size_t j_end = std::min(j + block_size, N);
                size_t k_end = std::min(k + block_size, K);
                
                for (size_t ii = i; ii < i_end; ++ii) {
                    for (size_t jj = j; jj < j_end; ++jj) {
                        T sum = C[ii * N + jj];
                        for (size_t kk = k; kk < k_end; ++kk) {
                            sum += A[ii * K + kk] * B[kk * N + jj];
                        }
                        C[ii * N + jj] = sum;
                    }
                }
            }
        }
    }
}
```

---

## 总结

本文档深入讲解了STL中的迭代器与算法技术，包括：

1. **迭代器深入剖析**：五种迭代器类型的完整实现、迭代器适配器
2. **算法设计与实现**：非修改序列、修改序列、排序、查找算法的底层实现
3. **函数对象与Lambda**：各种函数对象实现、Lambda编译器转换机制
4. **适配器与代理**：函数适配器、绑定机制
5. **性能优化技术**：循环展开、SIMD优化、缓存友好优化

这些技术展现了STL设计的精妙之处，掌握它们对深入理解算法库和编写高性能代码至关重要。