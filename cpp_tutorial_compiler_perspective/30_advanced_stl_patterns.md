# Advanced STL Patterns

## Introduction

Advanced STL patterns encompass sophisticated design techniques and usage patterns that leverage the full power of the Standard Template Library. These patterns go beyond basic container and algorithm usage to demonstrate how STL components can be combined to create elegant, efficient, and maintainable solutions. Understanding these patterns is essential for mastering modern C++ development.

## Custom Allocators and STL Integration

### Stateful Allocator Patterns

```cpp
// Advanced allocator patterns with stateful allocators
template<typename T>
class TrackingAllocator {
private:
    std::string name_;
    static std::atomic<std::size_t> total_allocated_;
    static std::atomic<std::size_t> total_deallocated_;
    
public:
    using value_type = T;
    
    explicit TrackingAllocator(const std::string& name = "default") : name_(name) {}
    
    template<typename U>
    TrackingAllocator(const TrackingAllocator<U>& other) noexcept : name_(other.name_) {}
    
    T* allocate(std::size_t n) {
        std::size_t size = n * sizeof(T);
        T* ptr = static_cast<T*>(std::malloc(size));
        if (!ptr) throw std::bad_alloc();
        
        total_allocated_ += size;
        std::cout << name_ << " allocator: Allocated " << size << " bytes" << std::endl;
        return ptr;
    }
    
    void deallocate(T* ptr, std::size_t n) {
        std::size_t size = n * sizeof(T);
        total_deallocated_ += size;
        std::cout << name_ << " allocator: Deallocated " << size << " bytes" << std::endl;
        std::free(ptr);
    }
    
    template<typename U>
    bool operator==(const TrackingAllocator<U>& other) const noexcept {
        return name_ == other.name_;
    }
    
    template<typename U>
    bool operator!=(const TrackingAllocator<U>& other) const noexcept {
        return !(*this == other);
    }
    
    static std::size_t get_total_allocated() { return total_allocated_.load(); }
    static std::size_t get_total_deallocated() { return total_deallocated_.load(); }
};

template<typename T>
std::atomic<std::size_t> TrackingAllocator<T>::total_allocated_{0};

template<typename T>
std::atomic<std::size_t> TrackingAllocator<T>::total_deallocated_{0};

// Pool allocator for frequently allocated objects
template<typename T, std::size_t BlockSize = 4096>
class PoolAllocator {
private:
    struct Block {
        alignas(T) char data[sizeof(T)];
        Block* next;
    };
    
    struct Pool {
        char* memory_pool;
        Block* free_list;
        std::size_t block_count;
        Pool* next_pool;
        
        Pool(std::size_t size) : next_pool(nullptr) {
            memory_pool = static_cast<char*>(std::malloc(size));
            if (!memory_pool) throw std::bad_alloc();
            
            block_count = size / sizeof(Block);
            Block* blocks = reinterpret_cast<Block*>(memory_pool);
            
            // Initialize free list
            free_list = blocks;
            for (std::size_t i = 0; i < block_count - 1; ++i) {
                blocks[i].next = &blocks[i + 1];
            }
            blocks[block_count - 1].next = nullptr;
        }
        
        ~Pool() {
            std::free(memory_pool);
        }
    };
    
    Pool* current_pool_;
    std::mutex pool_mutex_;
    
public:
    using value_type = T;
    
    PoolAllocator() : current_pool_(nullptr) {}
    
    template<typename U>
    PoolAllocator(const PoolAllocator<U, BlockSize>&) noexcept {}
    
    T* allocate(std::size_t n) {
        if (n != 1) {
            // Fall back to standard allocation for arrays
            return static_cast<T*>(std::malloc(n * sizeof(T)));
        }
        
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        if (!current_pool_ || !current_pool_->free_list) {
            // Create new pool
            Pool* new_pool = new Pool(BlockSize);
            new_pool->next_pool = current_pool_;
            current_pool_ = new_pool;
        }
        
        Block* block = current_pool_->free_list;
        current_pool_->free_list = block->next;
        return reinterpret_cast<T*>(block->data);
    }
    
    void deallocate(T* ptr, std::size_t n) {
        if (n != 1) {
            std::free(ptr);
            return;
        }
        
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        Block* block = reinterpret_cast<Block*>(reinterpret_cast<char*>(ptr) - offsetof(Block, data));
        block->next = current_pool_->free_list;
        current_pool_->free_list = block;
    }
    
    template<typename U>
    bool operator==(const PoolAllocator<U, BlockSize>&) const noexcept {
        return true;
    }
    
    template<typename U>
    bool operator!=(const PoolAllocator<U, BlockSize>&) const noexcept {
        return false;
    }
};

// Usage examples
void allocator_patterns_examples() {
    // Using tracking allocator
    std::vector<int, TrackingAllocator<int>> tracked_vector(TrackingAllocator<int>("VectorTracker"));
    tracked_vector.push_back(42);
    tracked_vector.push_back(100);
    
    std::cout << "Total allocated: " << TrackingAllocator<int>::get_total_allocated() << std::endl;
    std::cout << "Total deallocated: " << TrackingAllocator<int>::get_total_deallocated() << std::endl;
    
    // Using pool allocator
    std::vector<int, PoolAllocator<int>> pooled_vector;
    for (int i = 0; i < 100; ++i) {
        pooled_vector.push_back(i);
    }
}
```

## Custom Iterators and Iterator Adapters

### Iterator Adaptor Patterns

```cpp
// Advanced iterator patterns and adaptors
template<typename Iterator>
class StrideIterator {
private:
    Iterator current_;
    Iterator end_;
    std::ptrdiff_t stride_;
    
public:
    using iterator_category = typename std::iterator_traits<Iterator>::iterator_category;
    using value_type = typename std::iterator_traits<Iterator>::value_type;
    using difference_type = typename std::iterator_traits<Iterator>::difference_type;
    using pointer = typename std::iterator_traits<Iterator>::pointer;
    using reference = typename std::iterator_traits<Iterator>::reference;
    
    StrideIterator(Iterator current, Iterator end, std::ptrdiff_t stride)
        : current_(current), end_(end), stride_(stride) {}
    
    reference operator*() const { return *current_; }
    pointer operator->() const { return current_.operator->(); }
    
    StrideIterator& operator++() {
        std::advance(current_, stride_);
        if (current_ > end_) current_ = end_;
        return *this;
    }
    
    StrideIterator operator++(int) {
        StrideIterator temp = *this;
        ++(*this);
        return temp;
    }
    
    bool operator==(const StrideIterator& other) const { return current_ == other.current_; }
    bool operator!=(const StrideIterator& other) const { return !(*this == other); }
};

// Helper function to create stride iterator
template<typename Iterator>
auto make_stride_iterator(Iterator first, Iterator last, std::ptrdiff_t stride) {
    return StrideIterator<Iterator>(first, last, stride);
}

// Transform iterator adaptor
template<typename Iterator, typename Function>
class TransformIterator {
private:
    Iterator current_;
    Function func_;
    
public:
    using iterator_category = typename std::iterator_traits<Iterator>::iterator_category;
    using value_type = std::invoke_result_t<Function, typename std::iterator_traits<Iterator>::value_type>;
    using difference_type = typename std::iterator_traits<Iterator>::difference_type;
    using pointer = value_type*;
    using reference = value_type;
    
    TransformIterator(Iterator current, Function func)
        : current_(current), func_(std::move(func)) {}
    
    reference operator*() const { return func_(*current_); }
    
    TransformIterator& operator++() {
        ++current_;
        return *this;
    }
    
    TransformIterator operator++(int) {
        TransformIterator temp = *this;
        ++(*this);
        return temp;
    }
    
    bool operator==(const TransformIterator& other) const { return current_ == other.current_; }
    bool operator!=(const TransformIterator& other) const { return !(*this == other); }
};

// Helper function to create transform iterator
template<typename Iterator, typename Function>
auto make_transform_iterator(Iterator it, Function func) {
    return TransformIterator<Iterator, Function>(it, std::move(func));
}

// Usage examples
void iterator_adaptor_examples() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Stride iterator - process every 2nd element
    auto stride_begin = make_stride_iterator(vec.begin(), vec.end(), 2);
    auto stride_end = make_stride_iterator(vec.end(), vec.end(), 2);
    
    std::cout << "Every 2nd element: ";
    for (auto it = stride_begin; it != stride_end; ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // Transform iterator - square each element
    auto square_func = [](int x) { return x * x; };
    auto transform_begin = make_transform_iterator(vec.begin(), square_func);
    auto transform_end = make_transform_iterator(vec.end(), square_func);
    
    std::cout << "Squared elements: ";
    for (auto it = transform_begin; it != transform_end; ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // Combine both patterns
    auto combined_begin = make_transform_iterator(stride_begin, square_func);
    auto combined_end = make_transform_iterator(stride_end, square_func);
    
    std::cout << "Every 2nd element squared: ";
    for (auto it = combined_begin; it != combined_end; ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
}
```

## Function Objects and STL Algorithms

### Custom Function Objects

```cpp
// Advanced function object patterns
template<typename T>
class WeightedAverage {
private:
    std::vector<T> weights_;
    
public:
    explicit WeightedAverage(std::vector<T> weights) : weights_(std::move(weights)) {}
    
    template<typename Iterator>
    T operator()(Iterator first, Iterator last) const {
        if (weights_.empty()) return T{};
        
        T weighted_sum = T{};
        T total_weight = T{};
        auto weight_it = weights_.begin();
        
        for (auto it = first; it != last && weight_it != weights_.end(); ++it, ++weight_it) {
            weighted_sum += (*it) * (*weight_it);
            total_weight += *weight_it;
        }
        
        return total_weight != T{} ? weighted_sum / total_weight : T{};
    }
};

// Polymorphic function wrapper
template<typename Signature>
class FunctionWrapper;

template<typename R, typename... Args>
class FunctionWrapper<R(Args...)> {
private:
    struct Base {
        virtual ~Base() = default;
        virtual R call(Args... args) = 0;
    };
    
    template<typename F>
    struct Derived : Base {
        F func_;
        Derived(F func) : func_(std::move(func)) {}
        R call(Args... args) override { return func_(args...); }
    };
    
    std::unique_ptr<Base> impl_;
    
public:
    template<typename F>
    FunctionWrapper(F func) : impl_(std::make_unique<Derived<F>>(std::move(func))) {}
    
    R operator()(Args... args) { return impl_->call(args...); }
};

// Bind-like functionality with perfect forwarding
template<typename F, typename... BoundArgs>
class Binder {
private:
    F func_;
    std::tuple<BoundArgs...> bound_args_;
    
public:
    Binder(F func, BoundArgs... args) 
        : func_(std::move(func)), bound_args_(std::make_tuple(std::forward<BoundArgs>(args)...)) {}
    
    template<typename... CallArgs>
    auto operator()(CallArgs&&... args) {
        return std::apply(func_, std::tuple_cat(bound_args_, std::make_tuple(std::forward<CallArgs>(args)...)));
    }
};

template<typename F, typename... Args>
auto bind(F&& func, Args&&... args) {
    return Binder<std::decay_t<F>, std::decay_t<Args>...>(std::forward<F>(func), std::forward<Args>(args)...);
}

// Usage examples
void function_object_examples() {
    // Weighted average
    std::vector<double> values = {1.0, 2.0, 3.0, 4.0, 5.0};
    std::vector<double> weights = {0.1, 0.2, 0.3, 0.2, 0.2};
    
    WeightedAverage<double> weighted_avg(weights);
    double result = weighted_avg(values.begin(), values.end());
    std::cout << "Weighted average: " << result << std::endl;
    
    // Function wrapper
    FunctionWrapper<int(int, int)> wrapper([](int a, int b) { return a + b; });
    std::cout << "Wrapped function result: " << wrapper(3, 4) << std::endl;
    
    // Bind example
    auto multiply = [](int a, int b, int c) { return a * b * c; };
    auto bound_func = bind(multiply, 2, 3);  // Bind first two arguments
    std::cout << "Bound function result: " << bound_func(4) << std::endl;  // 2 * 3 * 4 = 24
}
```

## STL Container Adaptors and Custom Containers

### Advanced Container Patterns

```cpp
// Multi-index container pattern
template<typename T, typename... Indices>
class MultiIndexContainer {
private:
    std::vector<T> data_;
    std::tuple<std::map<typename Indices::key_type, std::size_t>...> indices_;
    
    template<std::size_t I>
    using index_type = std::tuple_element_t<I, std::tuple<Indices...>>;
    
public:
    void insert(const T& value) {
        std::size_t index = data_.size();
        data_.push_back(value);
        
        // Update all indices
        update_indices<0>(value, index);
    }
    
    template<typename Key>
    T* find(const Key& key) {
        return find_impl<0, Key>(key);
    }
    
private:
    template<std::size_t I>
    std::enable_if_t<(I < sizeof...(Indices)), void>
    update_indices(const T& value, std::size_t index) {
        using Index = index_type<I>;
        auto& map = std::get<I>(indices_);
        map[Index::get_key(value)] = index;
        update_indices<I + 1>(value, index);
    }
    
    template<std::size_t I>
    std::enable_if_t<(I >= sizeof...(Indices)), void>
    update_indices(const T& value, std::size_t index) {
        // Base case - do nothing
    }
    
    template<std::size_t I, typename Key>
    std::enable_if_t<(I < sizeof...(Indices)), T*>
    find_impl(const Key& key) {
        using Index = index_type<I>;
        if constexpr (std::is_same_v<typename Index::key_type, Key>) {
            auto& map = std::get<I>(indices_);
            auto it = map.find(key);
            if (it != map.end()) {
                return &data_[it->second];
            }
        }
        return find_impl<I + 1, Key>(key);
    }
    
    template<std::size_t I, typename Key>
    std::enable_if_t<(I >= sizeof...(Indices)), T*>
    find_impl(const Key& key) {
        return nullptr;  // Not found
    }
};

// Example index definitions
struct IdIndex {
    using key_type = int;
    template<typename T>
    static key_type get_key(const T& item) { return item.id; }
};

struct NameIndex {
    using key_type = std::string;
    template<typename T>
    static key_type get_key(const T& item) { return item.name; }
};

struct Person {
    int id;
    std::string name;
    int age;
};

// Circular buffer implementation
template<typename T, std::size_t Capacity>
class CircularBuffer {
private:
    std::array<T, Capacity> buffer_;
    std::size_t head_;
    std::size_t tail_;
    std::size_t size_;
    
public:
    CircularBuffer() : head_(0), tail_(0), size_(0) {}
    
    void push_back(const T& value) {
        buffer_[tail_] = value;
        tail_ = (tail_ + 1) % Capacity;
        if (size_ < Capacity) {
            ++size_;
        } else {
            head_ = (head_ + 1) % Capacity;
        }
    }
    
    T pop_front() {
        if (empty()) throw std::runtime_error("Buffer is empty");
        T value = buffer_[head_];
        head_ = (head_ + 1) % Capacity;
        --size_;
        return value;
    }
    
    const T& front() const {
        if (empty()) throw std::runtime_error("Buffer is empty");
        return buffer_[head_];
    }
    
    const T& back() const {
        if (empty()) throw std::runtime_error("Buffer is empty");
        std::size_t back_index = (tail_ == 0) ? Capacity - 1 : tail_ - 1;
        return buffer_[back_index];
    }
    
    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == Capacity; }
    std::size_t size() const { return size_; }
    std::size_t capacity() const { return Capacity; }
};

// Usage examples
void container_pattern_examples() {
    // Multi-index container
    MultiIndexContainer<Person, IdIndex, NameIndex> people;
    
    people.insert({1, "Alice", 30});
    people.insert({2, "Bob", 25});
    people.insert({3, "Charlie", 35});
    
    Person* found = people.find(2);  // Find by ID
    if (found) {
        std::cout << "Found person: " << found->name << std::endl;
    }
    
    found = people.find(std::string("Alice"));  // Find by name
    if (found) {
        std::cout << "Found person with ID: " << found->id << std::endl;
    }
    
    // Circular buffer
    CircularBuffer<int, 5> buffer;
    
    // Fill buffer
    for (int i = 1; i <= 7; ++i) {
        buffer.push_back(i);
        std::cout << "Pushed " << i << ", size: " << buffer.size() << std::endl;
    }
    
    // Pop elements
    while (!buffer.empty()) {
        std::cout << "Popped " << buffer.pop_front() << ", size: " << buffer.size() << std::endl;
    }
}
```

## STL Algorithm Composition Patterns

### Advanced Algorithm Patterns

```cpp
// Pipeline pattern for algorithm composition
template<typename Container>
class Pipeline {
private:
    Container data_;
    
public:
    explicit Pipeline(Container data) : data_(std::move(data)) {}
    
    template<typename Predicate>
    Pipeline filter(Predicate pred) && {
        Container filtered;
        std::copy_if(data_.begin(), data_.end(), std::back_inserter(filtered), pred);
        return Pipeline(std::move(filtered));
    }
    
    template<typename Function>
    auto transform(Function func) && {
        using result_type = std::decay_t<decltype(func(*data_.begin()))>;
        std::vector<result_type> transformed;
        transformed.reserve(data_.size());
        
        std::transform(data_.begin(), data_.end(), std::back_inserter(transformed), func);
        return Pipeline(std::move(transformed));
    }
    
    template<typename Comparator = std::less<>>
    Pipeline sort(Comparator comp = Comparator{}) && {
        std::sort(data_.begin(), data_.end(), comp);
        return std::move(*this);
    }
    
    template<typename Function>
    void for_each(Function func) && {
        std::for_each(data_.begin(), data_.end(), func);
    }
    
    Container collect() && {
        return std::move(data_);
    }
    
    std::size_t size() const { return data_.size(); }
};

// Helper function to create pipeline
template<typename Container>
auto make_pipeline(Container&& container) {
    return Pipeline(std::forward<Container>(container));
}

// Lazy evaluation pattern
template<typename Iterator, typename Function>
class LazyTransformRange {
private:
    Iterator first_;
    Iterator last_;
    Function func_;
    
public:
    LazyTransformRange(Iterator first, Iterator last, Function func)
        : first_(first), last_(last), func_(std::move(func)) {}
    
    class Iterator {
    private:
        typename LazyTransformRange::Iterator current_;
        Function func_;
        
    public:
        using iterator_category = std::input_iterator_tag;
        using value_type = std::invoke_result_t<Function, typename std::iterator_traits<Iterator>::value_type>;
        using difference_type = std::ptrdiff_t;
        using pointer = value_type*;
        using reference = value_type;
        
        Iterator(typename LazyTransformRange::Iterator current, Function func)
            : current_(current), func_(std::move(func)) {}
        
        reference operator*() const { return func_(*current_); }
        
        Iterator& operator++() { ++current_; return *this; }
        Iterator operator++(int) { Iterator temp = *this; ++(*this); return temp; }
        
        bool operator==(const Iterator& other) const { return current_ == other.current_; }
        bool operator!=(const Iterator& other) const { return !(*this == other); }
    };
    
    Iterator begin() const { return Iterator(first_, func_); }
    Iterator end() const { return Iterator(last_, func_); }
};

// Helper function for lazy transform
template<typename Iterator, typename Function>
auto make_lazy_transform(Iterator first, Iterator last, Function func) {
    return LazyTransformRange<Iterator, Function>(first, last, std::move(func));
}

// Usage examples
void algorithm_pattern_examples() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Pipeline example
    auto result = make_pipeline(numbers)
        .filter([](int x) { return x % 2 == 0; })  // Even numbers
        .transform([](int x) { return x * x; })    // Square them
        .sort()                                    // Sort
        .collect();                                // Collect result
    
    std::cout << "Pipeline result: ";
    for (int x : result) {
        std::cout << x << " ";
    }
    std::cout << std::endl;
    
    // Lazy evaluation example
    auto lazy_range = make_lazy_transform(numbers.begin(), numbers.end(), 
                                         [](int x) { return x * x; });
    
    std::cout << "Lazy transform (first 5): ";
    int count = 0;
    for (auto it = lazy_range.begin(); it != lazy_range.end() && count < 5; ++it, ++count) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
}
```

## Type Traits and Metaprogramming Patterns

### Advanced Type Trait Patterns

```cpp
// Compile-time type detection and manipulation
template<typename T>
struct is_iterable {
private:
    template<typename U>
    static auto test(int) -> decltype(
        std::declval<U>().begin(),
        std::declval<U>().end(),
        std::true_type{}
    );
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(0))::value;
};

template<typename T>
constexpr bool is_iterable_v = is_iterable<T>::value;

// Type list operations
template<typename... Types>
struct type_list {};

// Get type at index
template<std::size_t Index, typename List>
struct type_at;

template<std::size_t Index, typename Head, typename... Tail>
struct type_at<Index, type_list<Head, Tail...>> {
    using type = typename std::conditional_t<
        Index == 0,
        std::type_identity<Head>,
        type_at<Index - 1, type_list<Tail...>>
    >::type;
};

template<std::size_t Index>
struct type_at<Index, type_list<>> {
    static_assert(Index != Index, "Index out of bounds");
};

template<std::size_t Index, typename List>
using type_at_t = typename type_at<Index, List>::type;

// Find type in list
template<typename T, typename List>
struct contains;

template<typename T>
struct contains<T, type_list<>> : std::false_type {};

template<typename T, typename... Tail>
struct contains<T, type_list<T, Tail...>> : std::true_type {};

template<typename T, typename Head, typename... Tail>
struct contains<T, type_list<Head, Tail...>> : contains<T, type_list<Tail...>> {};

template<typename T, typename List>
constexpr bool contains_v = contains<T, List>::value;

// Conditional container selection
template<typename T>
using optimal_container = std::conditional_t<
    std::is_arithmetic_v<T>,
    std::vector<T>,
    std::list<T>
>;

// Usage examples
void metaprogramming_pattern_examples() {
    static_assert(is_iterable_v<std::vector<int>>);
    static_assert(!is_iterable_v<int>);
    
    using my_types = type_list<int, double, std::string>;
    static_assert(std::is_same_v<type_at_t<0, my_types>, int>);
    static_assert(std::is_same_v<type_at_t<1, my_types>, double>);
    static_assert(contains_v<std::string, my_types>);
    static_assert(!contains_v<char, my_types>);
    
    optimal_container<int> int_container;        // std::vector<int>
    optimal_container<std::string> string_container; // std::list<std::string>
    
    std::cout << "Type traits patterns demonstrated" << std::endl;
}
```

## Best Practices

### Advanced STL Pattern Guidelines

1. **Leverage Iterator Categories** - Use the most appropriate iterator category for your algorithms
2. **Combine STL Components** - Create powerful abstractions by combining containers, algorithms, and iterators
3. **Use Custom Allocators Selectively** - Only when you have specific memory management requirements
4. **Implement Lazy Evaluation** - For expensive operations that might not be needed
5. **Prefer Composition over Inheritance** - Combine STL components rather than inheriting from them
6. **Use Type Traits for Conditional Logic** - Enable compile-time decisions based on type properties
7. **Consider Performance Implications** - Profile your patterns with real data
8. **Maintain STL-Compatible Interfaces** - Ensure your custom components work seamlessly with STL algorithms

### Common Pattern Pitfalls

```cpp
// Examples of common pattern mistakes to avoid

// 1. Inefficient iterator usage
class PatternMistakes {
public:
    // Wrong: Inefficient random access on list
    void inefficient_access() {
        std::list<int> lst = {1, 2, 3, 4, 5};
        
        // O(n) operation repeated n times = O(n²)
        // for (std::size_t i = 0; i < lst.size(); ++i) {
        //     std::cout << *std::next(lst.begin(), i) << " ";  // Inefficient!
        // }
        
        // Correct: Use iterator-based algorithms
        for (const auto& item : lst) {
            std::cout << item << " ";
        }
    }
    
    // 2. Unnecessary copying in pipelines
    void pipeline_mistakes() {
        std::vector<std::string> strings = {"hello", "world", "cpp"};
        
        // Wrong: Creates unnecessary copies
        // auto result = strings 
        //     | std::views::filter([](const std::string& s) { return s.length() > 3; })
        //     | std::views::transform([](std::string s) { return std::move(s); });  // Unnecessary move
        
        // Correct: Use references where possible
        auto result = strings 
            | std::views::filter([](const std::string& s) { return s.length() > 3; })
            | std::views::transform([](const std::string& s) { return s; });
    }
    
    // 3. Thread-unsafe custom components
    class ThreadUnsafeCounter {
        static int counter_;  // Not thread-safe!
    public:
        static int get_next() { return ++counter_; }
    };
    
    // Correct: Thread-safe version
    class ThreadSafeCounter {
        static std::atomic<int> counter_;
    public:
        static int get_next() { return ++counter_; }
    };
};

int PatternMistakes::ThreadUnsafeCounter::counter_ = 0;
std::atomic<int> PatternMistakes::ThreadSafeCounter::counter_{0};

Advanced STL patterns provide powerful tools for creating elegant, efficient, and maintainable C++ code by leveraging the full capabilities of the Standard Template Library.