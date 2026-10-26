# Container Implementation Patterns: A Compiler's Perspective

## Introduction

STL containers follow specific implementation patterns that balance efficiency, safety, and genericity. Understanding how the compiler processes these patterns is essential for comprehending complex container code and designing efficient data structures.

## Base-Class Optimization Pattern

Containers often use inheritance for stateless components to save memory:

```cpp
// Empty Base Optimization (EBO)
template<typename T, typename Allocator = std::allocator<T>>
class vector {
private:
    // Inherit from allocator to save space when it's empty
    struct vector_impl : Allocator {
        T* data_;
        size_t size_;
        size_t capacity_;
        
        vector_impl(const Allocator& alloc = Allocator())
            : Allocator(alloc), data_(nullptr), size_(0), capacity_(0) {}
    };
    
    vector_impl impl_;
    
public:
    using allocator_type = Allocator;
    
    explicit vector(const Allocator& alloc = Allocator())
        : impl_(alloc) {}
    
    allocator_type get_allocator() const noexcept {
        return static_cast<allocator_type>(impl_);
    }
};
```

### Compiler's EBO Processing

The compiler optimizes memory layout through:

1. **Empty Base Detection**: Recognizes stateless allocators
2. **Layout Optimization**: Merges empty base with derived class
3. **Size Reduction**: Eliminates overhead of empty allocator member
4. **Alignment Preservation**: Maintains proper alignment requirements

## Template Specialization Patterns

Containers use specialization for optimization:

```cpp
// Primary template
template<typename T, typename Allocator = std::allocator<T>>
class vector {
    // General implementation
};

// Specialization for bool - space optimization
template<typename Allocator>
class vector<bool, Allocator> {
    // Bit-packed implementation
private:
    struct bit_reference {
        uint8_t* ptr_;
        uint8_t mask_;
        
        bit_reference(uint8_t* ptr, uint8_t mask)
            : ptr_(ptr), mask_(mask) {}
        
        operator bool() const noexcept {
            return (*ptr_ & mask_) != 0;
        }
        
        bit_reference& operator=(bool value) noexcept {
            if (value) {
                *ptr_ |= mask_;
            } else {
                *ptr_ &= ~mask_;
            }
            return *this;
        }
    };
    
    uint8_t* data_;
    size_t size_;
    size_t capacity_;
    
public:
    class iterator {
        uint8_t* ptr_;
        uint8_t mask_;
        
    public:
        iterator(uint8_t* ptr, uint8_t mask)
            : ptr_(ptr), mask_(mask) {}
        
        bit_reference operator*() const {
            return bit_reference(ptr_, mask_);
        }
        
        iterator& operator++() {
            if (mask_ == 0x80) {  // Last bit in byte
                ++ptr_;
                mask_ = 0x01;
            } else {
                mask_ <<= 1;
            }
            return *this;
        }
    };
};
```

### Compiler's Specialization Process

For `vector<bool>`:

1. **Template Matching**: Selects specialized implementation
2. **Type Transformation**: Converts `bool` elements to bits
3. **Iterator Adaptation**: Creates proxy reference type
4. **Memory Optimization**: 8x space reduction

## Iterator Implementation Patterns

Containers implement iterators with specific patterns:

```cpp
// Vector iterator implementation
template<typename T>
class vector_iterator {
private:
    T* ptr_;
    
public:
    using iterator_category = std::random_access_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using reference = T&;
    
    // Constructor
    explicit vector_iterator(T* ptr) : ptr_(ptr) {}
    
    // Dereference
    reference operator*() const noexcept { return *ptr_; }
    pointer operator->() const noexcept { return ptr_; }
    
    // Arithmetic operations
    vector_iterator& operator++() noexcept {
        ++ptr_;
        return *this;
    }
    
    vector_iterator operator++(int) noexcept {
        vector_iterator tmp = *this;
        ++ptr_;
        return tmp;
    }
    
    vector_iterator& operator--() noexcept {
        --ptr_;
        return *this;
    }
    
    // Random access operations
    vector_iterator& operator+=(difference_type n) noexcept {
        ptr_ += n;
        return *this;
    }
    
    vector_iterator operator+(difference_type n) const noexcept {
        return vector_iterator(ptr_ + n);
    }
    
    friend vector_iterator operator+(difference_type n, const vector_iterator& it) {
        return vector_iterator(it.ptr_ + n);
    }
    
    // Comparison operations
    bool operator==(const vector_iterator& other) const noexcept {
        return ptr_ == other.ptr_;
    }
    
    bool operator!=(const vector_iterator& other) const noexcept {
        return ptr_ != other.ptr_;
    }
    
    bool operator<(const vector_iterator& other) const noexcept {
        return ptr_ < other.ptr_;
    }
};
```

### Compiler's Iterator Optimization

The compiler optimizes iterators through:

1. **Inline Expansion**: All simple operations inlined
2. **Pointer Arithmetic**: Direct pointer manipulation
3. **Template Instantiation**: Type-specific optimized code
4. **Dead Code Elimination**: Unused operators removed

## Exception Safety Patterns

Containers implement different exception safety guarantees:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
public:
    // Strong exception safety guarantee
    void push_back(const T& value) {
        if (size_ >= capacity_) {
            // Strong guarantee: if reserve throws, *this is unchanged
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        // No-throw operation
        alloc_traits::construct(alloc_, data_ + size_, value);
        ++size_;
    }
    
    // Strong exception safety guarantee with move
    void push_back(T&& value) {
        if (size_ >= capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        alloc_traits::construct(alloc_, data_ + size_, std::move(value));
        ++size_;
    }
    
    // Basic exception safety guarantee
    template<typename... Args>
    void emplace_back(Args&&... args) {
        if (size_ >= capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        // If construct throws, size_ unchanged, strong guarantee
        alloc_traits::construct(alloc_, data_ + size_, std::forward<Args>(args)...);
        ++size_;
    }
    
    // No-throw guarantee
    void pop_back() noexcept {
        if (size_ > 0) {
            alloc_traits::destroy(alloc_, data_ + size_ - 1);
            --size_;
        }
    }
    
private:
    // Strong exception safety guarantee
    void reserve(size_t new_capacity) {
        if (new_capacity <= capacity_) return;
        
        // Allocate new memory - might throw
        T* new_data = alloc_traits::allocate(alloc_, new_capacity);
        
        // Move elements - might throw for non-noexcept move constructors
        size_t i = 0;
        try {
            for (; i < size_; ++i) {
                alloc_traits::construct(alloc_, new_data + i, std::move(data_[i]));
            }
        } catch (...) {
            // Cleanup constructed elements
            for (size_t j = 0; j < i; ++j) {
                alloc_traits::destroy(alloc_, new_data + j);
            }
            // Deallocate new memory
            alloc_traits::deallocate(alloc_, new_data, new_capacity);
            throw;  // Re-throw original exception
        }
        
        // Cleanup old elements and deallocate
        for (size_t i = 0; i < size_; ++i) {
            alloc_traits::destroy(alloc_, data_ + i);
        }
        alloc_traits::deallocate(alloc_, data_, capacity_);
        
        // Update pointers - no-throw
        data_ = new_data;
        capacity_ = new_capacity;
    }
};
```

### Compiler's Exception Handling

The compiler processes exception safety through:

1. **RAII Principles**: Automatic resource management
2. **Try-Catch Blocks**: Explicit exception handling where needed
3. **No-Throw Operations**: Marked with `noexcept`
4. **Strong Guarantee**: Careful ordering of operations

## Move-Only Container Patterns

Some containers work with move-only types:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
    static_assert(std::is_move_constructible<T>::value,
                  "T must be move constructible");
    
public:
    // Enable only for moveable types
    template<typename U = T>
    typename std::enable_if<std::is_move_constructible<U>::value>::type
    push_back(U&& value) {
        if (size_ >= capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        alloc_traits::construct(alloc_, data_ + size_, std::forward<U>(value));
        ++size_;
    }
    
    // Disable copy operations for move-only types
    template<typename U = T>
    typename std::enable_if<!std::is_copy_constructible<U>::value, vector&>::type
    operator=(const vector&) = delete;
    
    template<typename U = T>
    typename std::enable_if<!std::is_copy_constructible<U>::value>::type
    assign(size_t count, const T& value) = delete;
};
```

### Compiler's Move-Only Enforcement

The compiler enforces move-only constraints through:

1. **Static Assertions**: Compile-time type checking
2. **SFINAE**: Enable/disable functions based on type properties
3. **Deleted Functions**: Explicit disable of copy operations
4. **Concept Checking**: Validation of template requirements

## Container Swap Patterns

Efficient swapping is crucial for containers:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
public:
    void swap(vector& other) noexcept(
        std::allocator_traits<Allocator>::is_always_equal::value ||
        std::allocator_traits<Allocator>::propagate_on_container_swap::value) {
        
        using std::swap;
        
        // Always swap data pointers and sizes
        swap(data_, other.data_);
        swap(size_, other.size_);
        swap(capacity_, other.capacity_);
        
        // Conditionally swap allocators
        if constexpr (alloc_traits::propagate_on_container_swap::value) {
            swap(alloc_, other.alloc_);
        }
        // If allocators don't propagate and aren't equal, behavior is undefined
    }
    
    friend void swap(vector& lhs, vector& rhs) noexcept(noexcept(lhs.swap(rhs))) {
        lhs.swap(rhs);
    }
};
```

### Compiler's Swap Optimization

The compiler optimizes swapping through:

1. **Conditional noexcept**: Compile-time exception specification
2. **ADL Integration**: Argument-dependent lookup support
3. **Member vs. Free Functions**: Both interfaces provided
4. **Allocator Handling**: Proper allocator swapping based on traits

## Container Invalidation Patterns

Containers document iterator invalidation carefully:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
public:
    // Operations that invalidate iterators:
    // push_back: only if reallocation occurs
    // insert: all iterators and references after insertion point
    // erase: all iterators and references after erase point
    // resize: if container shrinks, all iterators after new end
    // clear: all iterators
    
    void push_back(const T& value) {
        // Implementation that may reallocate
        if (size_ >= capacity_) {
            // All iterators invalidated here
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        // No iterators invalidated here
        alloc_traits::construct(alloc_, data_ + size_, value);
        ++size_;
    }
    
    iterator insert(const_iterator pos, const T& value) {
        // Convert const_iterator to iterator
        size_t offset = pos - begin();
        
        // All iterators after pos invalidated
        if (size_ >= capacity_) {
            // All iterators invalidated
            reserve_insert(offset, value);
            return begin() + offset;
        }
        
        // Shift elements - iterators after pos invalidated
        iterator pos_it = begin() + offset;
        if (pos_it != end()) {
            // Move last element to new position
            alloc_traits::construct(alloc_, data_ + size_, std::move(data_[size_ - 1]));
            ++size_;
            // Move remaining elements
            std::move_backward(pos_it, end() - 2, end() - 1);
            *pos_it = value;
        } else {
            // Insert at end
            alloc_traits::construct(alloc_, data_ + size_, value);
            ++size_;
        }
        return pos_it;
    }
};
```

### Compiler's Invalidation Tracking

The compiler helps with invalidation through:

1. **Type Safety**: Iterator types tied to container types
2. **Lifetime Management**: RAII for iterator validity
3. **Debug Support**: Iterator debugging in debug builds
4. **Documentation**: Clear invalidation contracts

## Container Policy-Based Design

Advanced containers use policy-based design:

```cpp
// Policy-based container design
template<typename T,
         typename Allocator = std::allocator<T>,
         typename GrowthPolicy = default_growth_policy,
         typename StoragePolicy = default_storage_policy<T>>
class flexible_vector {
private:
    using storage_type = typename StoragePolicy::template storage<T, Allocator>;
    storage_type storage_;
    
public:
    void reserve(size_t new_capacity) {
        // Growth policy determines new capacity
        size_t actual_capacity = GrowthPolicy::calculate_capacity(
            storage_.capacity(), new_capacity);
        storage_.reallocate(actual_capacity);
    }
};

// Growth policy example
struct exponential_growth_policy {
    static size_t calculate_capacity(size_t current, size_t requested) {
        if (requested <= current) return current;
        size_t new_capacity = current == 0 ? 1 : current * 2;
        while (new_capacity < requested) {
            new_capacity *= 2;
        }
        return new_capacity;
    }
};

struct linear_growth_policy {
    static size_t calculate_capacity(size_t current, size_t requested) {
        if (requested <= current) return current;
        return std::max(requested, current + 64);  // Linear growth with minimum step
    }
};
```

### Compiler's Policy Processing

The compiler processes policies through:

1. **Template Composition**: Multiple policy parameters
2. **Policy Instantiation**: Each policy specialized for container
3. **Inline Optimization**: Policy functions inlined
4. **Zero-Overhead**: Policy abstraction with no runtime cost

## Container Interface Patterns

STL containers follow consistent interface patterns:

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class container {
public:
    // Type aliases
    using value_type = T;
    using allocator_type = Allocator;
    using size_type = typename std::allocator_traits<Allocator>::size_type;
    using difference_type = typename std::allocator_traits<Allocator>::difference_type;
    using reference = value_type&;
    using const_reference = const value_type&;
    using pointer = typename std::allocator_traits<Allocator>::pointer;
    using const_pointer = typename std::allocator_traits<Allocator>::const_pointer;
    
    // Iterator types (container-specific)
    using iterator = /* implementation-defined */;
    using const_iterator = /* implementation-defined */;
    using reverse_iterator = std::reverse_iterator<iterator>;
    using const_reverse_iterator = std::reverse_iterator<const_iterator>;
    
    // Constructors
    explicit container(const Allocator& alloc = Allocator());
    container(size_type count, const T& value, const Allocator& alloc = Allocator());
    container(const container& other);
    container(container&& other) noexcept;
    container(std::initializer_list<T> init, const Allocator& alloc = Allocator());
    
    // Assignment
    container& operator=(const container& other);
    container& operator=(container&& other) noexcept;
    container& operator=(std::initializer_list<T> ilist);
    
    // Allocator
    allocator_type get_allocator() const noexcept;
    
    // Element access
    reference at(size_type pos);
    const_reference at(size_type pos) const;
    reference operator[](size_type pos);
    const_reference operator[](size_type pos) const;
    reference front();
    const_reference front() const;
    reference back();
    const_reference back() const;
    
    // Iterators
    iterator begin() noexcept;
    const_iterator begin() const noexcept;
    const_iterator cbegin() const noexcept;
    iterator end() noexcept;
    const_iterator end() const noexcept;
    const_iterator cend() const noexcept;
    reverse_iterator rbegin() noexcept;
    const_reverse_iterator rbegin() const noexcept;
    const_reverse_iterator crbegin() const noexcept;
    reverse_iterator rend() noexcept;
    const_reverse_iterator rend() const noexcept;
    const_reverse_iterator crend() const noexcept;
    
    // Capacity
    bool empty() const noexcept;
    size_type size() const noexcept;
    size_type max_size() const noexcept;
    size_type capacity() const noexcept;  // For sequence containers
    
    // Modifiers
    void clear() noexcept;
    iterator insert(const_iterator pos, const T& value);
    iterator insert(const_iterator pos, T&& value);
    iterator insert(const_iterator pos, size_type count, const T& value);
    template<typename InputIt>
    iterator insert(const_iterator pos, InputIt first, InputIt last);
    iterator insert(const_iterator pos, std::initializer_list<T> ilist);
    template<typename... Args>
    iterator emplace(const_iterator pos, Args&&... args);
    iterator erase(const_iterator pos);
    iterator erase(const_iterator first, const_iterator last);
    void push_back(const T& value);  // For sequence containers
    void push_back(T&& value);       // For sequence containers
    template<typename... Args>
    reference emplace_back(Args&&... args);  // For sequence containers
    void pop_back();  // For sequence containers
    void push_front(const T& value);  // For deque/list/forward_list
    void push_front(T&& value);       // For deque/list/forward_list
    template<typename... Args>
    reference emplace_front(Args&&... args);  // For deque/list/forward_list
    void pop_front();  // For deque/list/forward_list
    void resize(size_type count);
    void resize(size_type count, const value_type& value);
    
    // Operations (associative containers)
    iterator find(const key_type& key);
    size_type count(const key_type& key);
    std::pair<iterator, iterator> equal_range(const key_type& key);
    iterator lower_bound(const key_type& key);
    iterator upper_bound(const key_type& key);
    
    // List operations (list/forward_list)
    void merge(list& other);
    void merge(list&& other);
    template<typename Compare>
    void merge(list& other, Compare comp);
    void splice(const_iterator pos, list& other);
    void splice(const_iterator pos, list&& other);
    void remove(const T& value);
    template<typename UnaryPredicate>
    void remove_if(UnaryPredicate p);
    void reverse() noexcept;
    void unique();
    template<typename BinaryPredicate>
    void unique(BinaryPredicate p);
    void sort();
    template<typename Compare>
    void sort(Compare comp);
};
```

### Compiler's Interface Processing

The compiler processes container interfaces through:

1. **Standard Compliance**: Required member functions and types
2. **Template Instantiation**: Type-specific implementations
3. **Overload Resolution**: Proper function selection
4. **Optimization**: Inlining and dead code elimination

## Understanding Complex STL Container Code

With container implementation knowledge, complex STL code becomes clearer:

```cpp
template<typename _Tp, typename _Alloc>
struct _Vector_base {
    typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template rebind<_Tp>::other _Tp_alloc_type;
    typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Alloc_traits;
    
    struct _Vector_impl_data {
        pointer _M_start;
        pointer _M_finish;
        pointer _M_end_of_storage;
    };
    
    struct _Vector_impl 
        : public _Tp_alloc_type, public _Vector_impl_data {
        _Vector_impl() _GLIBCXX_NOEXCEPT
            : _Tp_alloc_type() {}
        
        _Vector_impl(const _Tp_alloc_type& __a) _GLIBCXX_NOEXCEPT
            : _Tp_alloc_type(__a) {}
    };
    
public:
    _Vector_impl _M_impl;
    
    pointer _M_allocate(size_t __n) {
        typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Tr;
        return __n != 0 ? _Tr::allocate(_M_impl, __n) : pointer();
    }
    
    void _M_deallocate(pointer __p, size_t __n) {
        typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Tr;
        if (__p) _Tr::deallocate(_M_impl, __p, __n);
    }
};
```

The compiler:
1. **Uses EBO**: Inherits from allocator to save space
2. **Separates Concerns**: Data and allocator in different base classes
3. **Provides Interface**: Allocator traits for uniform access
4. **Manages Memory**: Allocation/deallocation through traits

## Summary

Key insights about container implementation patterns from a compiler perspective:

1. **EBO Optimization**: Empty base class optimization for stateless components
2. **Specialization**: Template specialization for type-specific optimizations
3. **Iterator Design**: Efficient iterator implementations with proper categories
4. **Exception Safety**: Strong, basic, and no-throw guarantees
5. **Move Semantics**: Support for move-only types and perfect forwarding
6. **Swap Operations**: Efficient swapping with proper allocator handling
7. **Invalidation Tracking**: Clear iterator invalidation contracts
8. **Policy Design**: Flexible design through policy-based templates
9. **Interface Consistency**: Standardized container interfaces

Understanding these patterns makes STL source code much more approachable. When you see:

```cpp
template<typename _Key, typename _Tp, typename _Compare, typename _Alloc>
class map {
    // Complex implementation using RB tree
};
```

You now understand that the compiler is:
1. Using template specialization for different key/value types
2. Applying comparison policies for ordering
3. Managing memory through allocator abstraction
4. Implementing efficient tree operations with proper iterator invalidation
5. Providing standard container interface with all required operations

This knowledge is essential for understanding how the STL provides both genericity and efficiency through sophisticated implementation patterns.