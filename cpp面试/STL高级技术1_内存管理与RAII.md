# STL高级技术（一）：内存管理与RAII

## 目录
1. [RAII原理与实现](#一raii原理与实现)
2. [智能指针技术](#二智能指针技术)
3. [分配器技术](#三分配器技术)
4. [内存池技术](#四内存池技术)
5. [迭代器失效与稳定性](#五迭代器失效与稳定性)

---

## 一、RAII原理与实现

### 1.1 RAII核心概念

RAII（Resource Acquisition Is Initialization，资源获取即初始化）是C++的核心惯用法，其核心思想是：

**资源的生命周期与对象的生命周期绑定，在构造函数中获取资源，在析构函数中释放资源。**

### 1.2 RAII的底层实现原理

```cpp
// RAII的基本实现模式
class ResourceHandler {
public:
    // 构造函数：获取资源
    ResourceHandler(const char* resource_name) 
        : resource_(open_resource(resource_name)) {
        if (!resource_) {
            throw std::runtime_error("Failed to acquire resource");
        }
    }
    
    // 析构函数：自动释放资源
    ~ResourceHandler() {
        if (resource_) {
            close_resource(resource_);
        }
    }
    
    // 禁止拷贝（防止资源重复释放）
    ResourceHandler(const ResourceHandler&) = delete;
    ResourceHandler& operator=(const ResourceHandler&) = delete;
    
    // 允许移动（转移资源所有权）
    ResourceHandler(ResourceHandler&& other) noexcept
        : resource_(other.resource_) {
        other.resource_ = nullptr;
    }
    
    ResourceHandler& operator=(ResourceHandler&& other) noexcept {
        if (this != &other) {
            if (resource_) {
                close_resource(resource_);
            }
            resource_ = other.resource_;
            other.resource_ = nullptr;
        }
        return *this;
    }
    
private:
    void* resource_;
};

// 模拟资源操作
void* open_resource(const char* name) {
    std::cout << "Opening resource: " << name << std::endl;
    return reinterpret_cast<void*>(0x12345678);
}

void close_resource(void* resource) {
    std::cout << "Closing resource" << std::endl;
}
```

### 1.3 RAII在STL中的深度应用

#### 1.3.1 vector的内存管理

```cpp
// vector的简化实现，展示RAII在动态数组中的应用
template<typename T, typename Allocator = std::allocator<T>>
class my_vector {
public:
    using allocator_type = Allocator;
    using value_type = T;
    using size_type = std::size_t;
    using pointer = T*;
    
    // 默认构造函数
    my_vector() noexcept(noexcept(Allocator()))
        : allocator_(), data_(nullptr), size_(0), capacity_(0) {}
    
    // 带分配器的构造函数
    explicit my_vector(const Allocator& alloc) noexcept
        : allocator_(alloc), data_(nullptr), size_(0), capacity_(0) {}
    
    // 指定大小的构造函数
    explicit my_vector(size_type count, const T& value = T(), 
                       const Allocator& alloc = Allocator())
        : allocator_(alloc), data_(nullptr), size_(0), capacity_(0) {
        resize(count, value);
    }
    
    // 初始化列表构造函数
    my_vector(std::initializer_list<T> init, const Allocator& alloc = Allocator())
        : allocator_(alloc), data_(nullptr), size_(0), capacity_(0) {
        reserve(init.size());
        for (const auto& elem : init) {
            push_back(elem);
        }
    }
    
    // 析构函数：RAII的核心 - 自动释放内存
    ~my_vector() {
        clear();
        deallocate();
    }
    
    // 拷贝构造函数
    my_vector(const my_vector& other)
        : allocator_(std::allocator_traits<Allocator>::select_on_container_copy_construction(other.allocator_)),
          data_(nullptr), size_(0), capacity_(0) {
        reserve(other.size_);
        for (size_type i = 0; i < other.size_; ++i) {
            emplace_back(other.data_[i]);
        }
    }
    
    // 移动构造函数
    my_vector(my_vector&& other) noexcept
        : allocator_(std::move(other.allocator_)),
          data_(other.data_), size_(other.size_), capacity_(other.capacity_) {
        other.data_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }
    
    // 拷贝赋值运算符
    my_vector& operator=(const my_vector& other) {
        if (this != &other) {
            clear();
            if (capacity_ < other.size_) {
                deallocate();
                reserve(other.size_);
            }
            for (size_type i = 0; i < other.size_; ++i) {
                emplace_back(other.data_[i]);
            }
        }
        return *this;
    }
    
    // 移动赋值运算符
    my_vector& operator=(my_vector&& other) noexcept {
        if (this != &other) {
            clear();
            deallocate();
            
            allocator_ = std::move(other.allocator_);
            data_ = other.data_;
            size_ = other.size_;
            capacity_ = other.capacity_;
            
            other.data_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        }
        return *this;
    }
    
    // 元素访问
    reference operator[](size_type pos) {
        return data_[pos];
    }
    
    const_reference operator[](size_type pos) const {
        return data_[pos];
    }
    
    reference at(size_type pos) {
        if (pos >= size_) {
            throw std::out_of_range("Index out of range");
        }
        return data_[pos];
    }
    
    // 迭代器
    iterator begin() { return data_; }
    const_iterator begin() const { return data_; }
    const_iterator cbegin() const { return data_; }
    
    iterator end() { return data_ + size_; }
    const_iterator end() const { return data_ + size_; }
    const_iterator cend() const { return data_ + size_; }
    
    // 容量
    bool empty() const noexcept { return size_ == 0; }
    size_type size() const noexcept { return size_; }
    size_type capacity() const noexcept { return capacity_; }
    
    void reserve(size_type new_capacity) {
        if (new_capacity > capacity_) {
            pointer new_data = allocator_.allocate(new_capacity);
            
            // 移动现有元素
            for (size_type i = 0; i < size_; ++i) {
                std::allocator_traits<Allocator>::construct(
                    allocator_, new_data + i, std::move(data_[i])
                );
                std::allocator_traits<Allocator>::destroy(allocator_, data_ + i);
            }
            
            deallocate();
            data_ = new_data;
            capacity_ = new_capacity;
        }
    }
    
    void resize(size_type new_size, const T& value = T()) {
        if (new_size > size_) {
            reserve(new_size);
            for (size_type i = size_; i < new_size; ++i) {
                emplace_back(value);
            }
        } else if (new_size < size_) {
            for (size_type i = new_size; i < size_; ++i) {
                std::allocator_traits<Allocator>::destroy(allocator_, data_ + i);
            }
            size_ = new_size;
        }
    }
    
    void clear() noexcept {
        for (size_type i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::destroy(allocator_, data_ + i);
        }
        size_ = 0;
    }
    
    void shrink_to_fit() {
        if (size_ < capacity_) {
            pointer new_data = nullptr;
            if (size_ > 0) {
                new_data = allocator_.allocate(size_);
                for (size_type i = 0; i < size_; ++i) {
                    std::allocator_traits<Allocator>::construct(
                        allocator_, new_data + i, std::move(data_[i])
                    );
                    std::allocator_traits<Allocator>::destroy(allocator_, data_ + i);
                }
            }
            deallocate();
            data_ = new_data;
            capacity_ = size_;
        }
    }
    
    // 修改器
    void push_back(const T& value) {
        emplace_back(value);
    }
    
    void push_back(T&& value) {
        emplace_back(std::move(value));
    }
    
    template<typename... Args>
    void emplace_back(Args&&... args) {
        if (size_ == capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        std::allocator_traits<Allocator>::construct(
            allocator_, data_ + size_, std::forward<Args>(args)...
        );
        ++size_;
    }
    
    void pop_back() {
        if (size_ > 0) {
            --size_;
            std::allocator_traits<Allocator>::destroy(allocator_, data_ + size_);
        }
    }
    
    iterator insert(const_iterator pos, const T& value) {
        return emplace(pos, value);
    }
    
    iterator insert(const_iterator pos, T&& value) {
        return emplace(pos, std::move(value));
    }
    
    template<typename... Args>
    iterator emplace(const_iterator pos, Args&&... args) {
        size_type index = pos - data_;
        
        if (size_ == capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        
        // 移动元素
        for (size_type i = size_; i > index; --i) {
            std::allocator_traits<Allocator>::construct(
                allocator_, data_ + i, std::move(data_[i - 1])
            );
            std::allocator_traits<Allocator>::destroy(allocator_, data_ + i - 1);
        }
        
        std::allocator_traits<Allocator>::construct(
            allocator_, data_ + index, std::forward<Args>(args)...
        );
        ++size_;
        
        return data_ + index;
    }
    
    iterator erase(const_iterator pos) {
        size_type index = pos - data_;
        if (index < size_) {
            std::allocator_traits<Allocator>::destroy(allocator_, data_ + index);
            
            for (size_type i = index; i < size_ - 1; ++i) {
                std::allocator_traits<Allocator>::construct(
                    allocator_, data_ + i, std::move(data_[i + 1])
                );
                std::allocator_traits<Allocator>::destroy(allocator_, data_ + i + 1);
            }
            
            --size_;
        }
        return data_ + index;
    }
    
    // 分配器访问
    allocator_type get_allocator() const noexcept {
        return allocator_;
    }
    
private:
    void deallocate() {
        if (data_) {
            allocator_.deallocate(data_, capacity_);
            data_ = nullptr;
        }
    }
    
    allocator_type allocator_;
    pointer data_;
    size_type size_;
    size_type capacity_;
};
```

### 1.4 RAII的优势与面试要点

#### 面试问题1：RAII如何防止资源泄漏？

**答案：**
RAII通过对象的生命周期管理资源，利用C++的确定性析构机制保证资源释放：
1. 无论函数如何返回（正常返回、异常返回），析构函数都会被调用
2. 栈展开保证局部对象的析构函数被调用
3. 自动处理资源清理，无需手动管理

```cpp
// 示例：RAII防止资源泄漏
void function_with_exception() {
    // 手动管理（容易出错）
    FILE* file1 = fopen("test1.txt", "r");
    if (!file1) return;
    
    FILE* file2 = fopen("test2.txt", "r");
    if (!file2) {
        fclose(file1);  // 必须记得清理
        return;
    }
    
    // 如果这里抛出异常，file1和file2都不会被关闭
    // 导致资源泄漏
    
    fclose(file1);
    fclose(file2);
}

// RAII方式（自动清理）
class FileRAII {
    FILE* file_;
public:
    FileRAII(const char* filename, const char* mode) 
        : file_(fopen(filename, mode)) {
        if (!file_) throw std::runtime_error("Cannot open file");
    }
    
    ~FileRAII() {
        if (file_) fclose(file_);
    }
    
    // 禁止拷贝，允许移动
    FileRAII(const FileRAII&) = delete;
    FileRAII& operator=(const FileRAII&) = delete;
    
    FILE* get() const { return file_; }
};

void function_with_raii() {
    FileRAII file1("test1.txt", "r");
    FileRAII file2("test2.txt", "r");
    
    // 即使抛出异常，文件也会自动关闭
    throw std::runtime_error("Error occurred");
}  // 自动清理
```

#### 面试问题2：RAII在异常安全中的应用？

**答案：**
RAII是实现异常安全的关键技术：

1. **基本保证**：对象保持有效状态，无资源泄漏
2. **强保证**：操作要么完全成功，要么完全不执行
3. **不抛出保证**：操作永不抛出异常

```cpp
// 异常安全的栈实现
template<typename T>
class Stack {
public:
    void push(const T& value) {
        // 强保证：要么成功，要么不影响原有状态
        Stack temp(*this);  // 备份当前状态
        
        try {
            temp.data_.push_back(value);  // 可能抛出异常
            swap(temp);  // 只有成功才交换
        } catch (...) {
            // 异常发生，原有状态不受影响
        }
    }
    
private:
    std::vector<T> data_;
    
    void swap(Stack& other) noexcept {
        data_.swap(other.data_);
    }
};
```

---

## 二、智能指针技术

### 2.1 智能指针的底层实现机制

#### 2.1.1 unique_ptr实现

```cpp
// unique_ptr的简化实现
template<typename T, typename Deleter = std::default_delete<T>>
class unique_ptr {
public:
    using element_type = T;
    using deleter_type = Deleter;
    using pointer = typename std::add_pointer<T>::type;
    
    // 默认构造函数
    constexpr unique_ptr() noexcept
        : ptr_(nullptr), deleter_() {}
    
    constexpr unique_ptr(std::nullptr_t) noexcept
        : ptr_(nullptr), deleter_() {}
    
    // 显式构造
    explicit unique_ptr(pointer p) noexcept
        : ptr_(p), deleter_() {}
    
    unique_ptr(pointer p, const Deleter& d) noexcept
        : ptr_(p), deleter_(d) {}
    
    unique_ptr(pointer p, Deleter&& d) noexcept
        : ptr_(p), deleter_(std::move(d)) {}
    
    // 移动构造
    unique_ptr(unique_ptr&& other) noexcept
        : ptr_(other.ptr_), deleter_(std::move(other.deleter_)) {
        other.ptr_ = nullptr;
    }
    
    // 模板移动构造（支持类型转换）
    template<typename U, typename E>
    unique_ptr(unique_ptr<U, E>&& other) noexcept
        : ptr_(other.ptr_), deleter_(std::move(other.deleter_)) {
        other.ptr_ = nullptr;
    }
    
    // 析构函数
    ~unique_ptr() {
        reset();
    }
    
    // 移动赋值
    unique_ptr& operator=(unique_ptr&& other) noexcept {
        if (this != &other) {
            reset();
            ptr_ = other.ptr_;
            deleter_ = std::move(other.deleter_);
            other.ptr_ = nullptr;
        }
        return *this;
    }
    
    template<typename U, typename E>
    unique_ptr& operator=(unique_ptr<U, E>&& other) noexcept {
        if (this != &other) {
            reset();
            ptr_ = other.ptr_;
            deleter_ = std::move(other.deleter_);
            other.ptr_ = nullptr;
        }
        return *this;
    }
    
    // 赋值nullptr
    unique_ptr& operator=(std::nullptr_t) noexcept {
        reset();
        return *this;
    }
    
    // 禁止拷贝
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;
    
    // 解引用
    T& operator*() const {
        return *ptr_;
    }
    
    T* operator->() const noexcept {
        return ptr_;
    }
    
    // 获取原始指针
    pointer get() const noexcept {
        return ptr_;
    }
    
    // 释放所有权
    pointer release() noexcept {
        pointer temp = ptr_;
        ptr_ = nullptr;
        return temp;
    }
    
    // 重置
    void reset(pointer p = pointer()) noexcept {
        if (ptr_ != p) {
            if (ptr_) {
                deleter_(ptr_);
            }
            ptr_ = p;
        }
    }
    
    // 交换
    void swap(unique_ptr& other) noexcept {
        std::swap(ptr_, other.ptr_);
        std::swap(deleter_, other.deleter_);
    }
    
    // 显式bool转换
    explicit operator bool() const noexcept {
        return ptr_ != nullptr;
    }
    
    // 删除器访问
    Deleter& get_deleter() noexcept {
        return deleter_;
    }
    
    const Deleter& get_deleter() const noexcept {
        return deleter_;
    }
    
    // 数组特化支持
    T& operator[](size_t i) const {
        return ptr_[i];
    }
    
private:
    pointer ptr_;
    Deleter deleter_;
};

// unique_ptr工厂函数
template<typename T, typename... Args>
unique_ptr<T> make_unique(Args&&... args) {
    return unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 数组特化
template<typename T, typename... Args>
unique_ptr<T> make_unique(Args&&... args) = delete;

template<typename T>
unique_ptr<T> make_unique(std::size_t size) {
    return unique_ptr<T>(new typename std::remove_extent<T>::type[size]());
}
```

#### 2.1.2 shared_ptr实现

```cpp
// shared_ptr的简化实现
template<typename T>
class shared_ptr {
public:
    using element_type = T;
    
    // 控制块：管理引用计数
    struct control_block {
        std::atomic<size_t> ref_count;
        std::atomic<size_t> weak_count;
        
        control_block() : ref_count(1), weak_count(0) {}
        
        virtual ~control_block() {}
        
        virtual void destroy() = 0;
        virtual void delete_this() = 0;
    };
    
    template<typename U>
    struct control_block_impl : control_block {
        U* ptr;
        std::function<void(U*)> deleter;
        std::function<void(control_block*)> allocator_deleter;
        
        control_block_impl(U* p, 
                          std::function<void(U*)> d,
                          std::function<void(control_block*)> ad)
            : ptr(p), deleter(d), allocator_deleter(ad) {}
        
        void destroy() override {
            if (ptr && deleter) {
                deleter(ptr);
            }
            ptr = nullptr;
        }
        
        void delete_this() override {
            if (allocator_deleter) {
                allocator_deleter(this);
            } else {
                delete this;
            }
        }
    };
    
    // 默认构造
    constexpr shared_ptr() noexcept
        : ptr_(nullptr), ctrl_(nullptr) {}
    
    constexpr shared_ptr(std::nullptr_t) noexcept
        : ptr_(nullptr), ctrl_(nullptr) {}
    
    // 构造函数
    explicit shared_ptr(T* p) {
        if (p) {
            ctrl_ = new control_block_impl<T>(
                p, 
                [](T* ptr) { delete ptr; },
                nullptr
            );
            ptr_ = p;
        }
    }
    
    template<typename U>
    explicit shared_ptr(U* p) {
        if (p) {
            ctrl_ = new control_block_impl<U>(
                p,
                [](U* ptr) { delete ptr; },
                nullptr
            );
            ptr_ = p;
        }
    }
    
    // 带删除器的构造
    template<typename U, typename Deleter>
    shared_ptr(U* p, Deleter d) {
        if (p) {
            ctrl_ = new control_block_impl<U>(
                p,
                [d](U* ptr) { d(ptr); },
                nullptr
            );
            ptr_ = p;
        }
    }
    
    // 拷贝构造
    shared_ptr(const shared_ptr& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) {
            ctrl_->ref_count.fetch_add(1, std::memory_order_relaxed);
        }
    }
    
    template<typename U>
    shared_ptr(const shared_ptr<U>& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) {
            ctrl_->ref_count.fetch_add(1, std::memory_order_relaxed);
        }
    }
    
    // 移动构造
    shared_ptr(shared_ptr&& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        other.ptr_ = nullptr;
        other.ctrl_ = nullptr;
    }
    
    template<typename U>
    shared_ptr(shared_ptr<U>&& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        other.ptr_ = nullptr;
        other.ctrl_ = nullptr;
    }
    
    // 析构函数
    ~shared_ptr() {
        if (ctrl_) {
            if (ctrl_->ref_count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
                ctrl_->destroy();
                if (ctrl_->weak_count.load(std::memory_order_acquire) == 0) {
                    ctrl_->delete_this();
                }
            }
        }
    }
    
    // 拷贝赋值
    shared_ptr& operator=(const shared_ptr& other) noexcept {
        if (this != &other) {
            shared_ptr(other).swap(*this);
        }
        return *this;
    }
    
    template<typename U>
    shared_ptr& operator=(const shared_ptr<U>& other) noexcept {
        if (this != &other) {
            shared_ptr(other).swap(*this);
        }
        return *this;
    }
    
    // 移动赋值
    shared_ptr& operator=(shared_ptr&& other) noexcept {
        if (this != &other) {
            shared_ptr(std::move(other)).swap(*this);
        }
        return *this;
    }
    
    template<typename U>
    shared_ptr& operator=(shared_ptr<U>&& other) noexcept {
        if (this != &other) {
            shared_ptr(std::move(other)).swap(*this);
        }
        return *this;
    }
    
    // 赋值nullptr
    shared_ptr& operator=(std::nullptr_t) noexcept {
        reset();
        return *this;
    }
    
    // 解引用
    T& operator*() const noexcept {
        return *ptr_;
    }
    
    T* operator->() const noexcept {
        return ptr_;
    }
    
    // 获取原始指针
    T* get() const noexcept {
        return ptr_;
    }
    
    // 引用计数
    long use_count() const noexcept {
        return ctrl_ ? ctrl_->ref_count.load(std::memory_order_relaxed) : 0;
    }
    
    bool unique() const noexcept {
        return use_count() == 1;
    }
    
    // 显式bool转换
    explicit operator bool() const noexcept {
        return ptr_ != nullptr;
    }
    
    // 重置
    void reset() noexcept {
        shared_ptr().swap(*this);
    }
    
    template<typename U>
    void reset(U* p) {
        shared_ptr(p).swap(*this);
    }
    
    // 交换
    void swap(shared_ptr& other) noexcept {
        std::swap(ptr_, other.ptr_);
        std::swap(ctrl_, other.ctrl_);
    }
    
    // 所有权的获取
    T* owner_before(const shared_ptr& other) const noexcept {
        return ptr_ < other.ptr_;
    }
    
private:
    T* ptr_;
    control_block* ctrl_;
    
    template<typename U>
    friend class shared_ptr;
    
    template<typename U>
    friend class weak_ptr;
};

// weak_ptr的简化实现
template<typename T>
class weak_ptr {
public:
    using element_type = T;
    
    // 默认构造
    constexpr weak_ptr() noexcept
        : ptr_(nullptr), ctrl_(nullptr) {}
    
    // 拷贝构造
    weak_ptr(const weak_ptr& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) {
            ctrl_->weak_count.fetch_add(1, std::memory_order_relaxed);
        }
    }
    
    template<typename U>
    weak_ptr(const weak_ptr<U>& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) {
            ctrl_->weak_count.fetch_add(1, std::memory_order_relaxed);
        }
    }
    
    template<typename U>
    weak_ptr(const shared_ptr<U>& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) {
            ctrl_->weak_count.fetch_add(1, std::memory_order_relaxed);
        }
    }
    
    // 移动构造
    weak_ptr(weak_ptr&& other) noexcept
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        other.ptr_ = nullptr;
        other.ctrl_ = nullptr;
    }
    
    // 析构函数
    ~weak_ptr() {
        if (ctrl_) {
            if (ctrl_->weak_count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
                if (ctrl_->ref_count.load(std::memory_order_acquire) == 0) {
                    ctrl_->delete_this();
                }
            }
        }
    }
    
    // 拷贝赋值
    weak_ptr& operator=(const weak_ptr& other) noexcept {
        if (this != &other) {
            weak_ptr(other).swap(*this);
        }
        return *this;
    }
    
    template<typename U>
    weak_ptr& operator=(const weak_ptr<U>& other) noexcept {
        if (this != &other) {
            weak_ptr(other).swap(*this);
        }
        return *this;
    }
    
    // 移动赋值
    weak_ptr& operator=(weak_ptr&& other) noexcept {
        if (this != &other) {
            weak_ptr(std::move(other)).swap(*this);
        }
        return *this;
    }
    
    // lock：尝试获取shared_ptr
    shared_ptr<T> lock() const noexcept {
        shared_ptr<T> result;
        if (ctrl_ && ctrl_->ref_count.load(std::memory_order_acquire) > 0) {
            result.ptr_ = ptr_;
            result.ctrl_ = ctrl_;
            ctrl_->ref_count.fetch_add(1, std::memory_order_relaxed);
        }
        return result;
    }
    
    // 引用计数
    long use_count() const noexcept {
        return ctrl_ ? ctrl_->ref_count.load(std::memory_order_relaxed) : 0;
    }
    
    bool expired() const noexcept {
        return use_count() == 0;
    }
    
    // 重置
    void reset() noexcept {
        weak_ptr().swap(*this);
    }
    
    // 交换
    void swap(weak_ptr& other) noexcept {
        std::swap(ptr_, other.ptr_);
        std::swap(ctrl_, other.ctrl_);
    }
    
private:
    T* ptr_;
    typename shared_ptr<T>::control_block* ctrl_;
    
    template<typename U>
    friend class shared_ptr;
};

// make_shared工厂函数
template<typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args) {
    // 使用单个内存块分配控制块和对象
    struct block {
        typename shared_ptr<T>::control_block_impl<T> ctrl;
        typename std::aligned_storage<sizeof(T), alignof(T)>::type storage;
        
        block(Args&&... args) 
            : ctrl(nullptr, nullptr, nullptr) {
            new (&storage) T(std::forward<Args>(args)...);
            ctrl.ptr = reinterpret_cast<T*>(&storage);
            ctrl.deleter = [](T* p) {
                p->~T();
            };
            ctrl.allocator_deleter = [](typename shared_ptr<T>::control_block* cb) {
                delete static_cast<block*>(cb);
            };
        }
    };
    
    block* b = new block(std::forward<Args>(args)...);
    shared_ptr<T> result;
    result.ptr_ = b->ctrl.ptr;
    result.ctrl_ = &b->ctrl;
    
    return result;
}
```

### 2.2 智能指针的高级应用

#### 2.2.1 自定义删除器

```cpp
// 文件句柄的智能指针
struct FileDeleter {
    void operator()(FILE* file) const {
        if (file) {
            std::cout << "Closing file" << std::endl;
            fclose(file);
        }
    }
};

using FilePtr = std::unique_ptr<FILE, FileDeleter>;

FilePtr open_file(const char* filename, const char* mode) {
    FILE* file = fopen(filename, mode);
    if (!file) {
        throw std::runtime_error("Cannot open file");
    }
    return FilePtr(file);
}

// 数组删除器
struct ArrayDeleter {
    template<typename T>
    void operator()(T* ptr) const {
        delete[] ptr;
    }
};

template<typename T>
using UniqueArray = std::unique_ptr<T[], ArrayDeleter>;

UniqueArray<int> create_array(size_t size) {
    return UniqueArray<int>(new int[size]);
}

// Lambda删除器
auto custom_deleter = [](void* ptr) {
    std::cout << "Custom deleter called" << std::endl;
    free(ptr);
};

using CustomPtr = std::unique_ptr<void, decltype(custom_deleter)>;
```

#### 2.2.2 循环引用问题

```cpp
// 循环引用示例
class Node {
public:
    std::shared_ptr<Node> next;
    
    ~Node() {
        std::cout << "Node destroyed" << std::endl;
    }
};

// 创建循环引用
void create_cycle() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;
    node2->next = node1;  // 循环引用
    
    // node1和node2都不会被释放
}

// 使用weak_ptr解决循环引用
class Node2 {
public:
    std::shared_ptr<Node2> next;
    std::weak_ptr<Node2> prev;  // 使用weak_ptr
    
    ~Node2() {
        std::cout << "Node2 destroyed" << std::endl;
    }
};

void create_acyclic() {
    auto node1 = std::make_shared<Node2>();
    auto node2 = std::make_shared<Node2>();
    
    node1->next = node2;
    node2->prev = node1;  // weak_ptr不增加引用计数
    
    // 节点可以正常释放
}
```

### 2.3 智能指针面试问题

#### 面试问题3：shared_ptr的线程安全性如何？

**答案：**
shared_ptr的线程安全性：

1. **引用计数操作是原子操作**：线程安全
2. **控制块本身是线程安全的**
3. **指向对象的访问不是线程安全的**：需要外部同步

```cpp
// 正确的线程安全使用
std::shared_ptr<int> global_ptr = std::make_shared<int>(42);
std::mutex mtx;

// 线程1：安全的读取
void thread_safe_read() {
    std::shared_ptr<int> local_ptr;
    {
        std::lock_guard<std::mutex> lock(mtx);
        local_ptr = global_ptr;  // 拷贝是原子的
    }
    // 使用local_ptr，不需要锁
    std::cout << *local_ptr << std::endl;
}

// 线程2：安全的写入
void thread_safe_write(int new_value) {
    std::lock_guard<std::mutex> lock(mtx);
    global_ptr = std::make_shared<int>(new_value);
}
```

#### 面试问题4：enable_shared_from_this的原理是什么？

**答案：**
enable_shared_from_this允许在类成员函数中获取shared_ptr：

```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    std::shared_ptr<MyClass> get_shared() {
        return shared_from_this();  // 获取shared_ptr
    }
    
private:
    // enable_shared_from_this包含一个weak_ptr
    // 在shared_ptr构造时自动初始化
};

// 使用
auto obj = std::make_shared<MyClass>();
auto shared = obj->get_shared();  // 正确
```

---

## 三、分配器技术

### 3.1 分配器的设计原理

#### 3.1.1 标准分配器接口

```cpp
// 标准分配器的简化实现
template<typename T>
class Allocator {
public:
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
    
    // 模板构造函数
    template<typename U>
    struct rebind {
        using other = Allocator<U>;
    };
    
    // 默认构造函数
    constexpr Allocator() noexcept {}
    
    // 拷贝构造函数
    constexpr Allocator(const Allocator&) noexcept = default;
    
    // 模板拷贝构造函数
    template<typename U>
    constexpr Allocator(const Allocator<U>&) noexcept {}
    
    // 析构函数
    ~Allocator() = default;
    
    // 分配内存
    T* allocate(std::size_t n) {
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T)) {
            throw std::bad_alloc();
        }
        
        if (auto p = static_cast<T*>(std::malloc(n * sizeof(T)))) {
            return p;
        }
        
        throw std::bad_alloc();
    }
    
    // 释放内存
    void deallocate(T* p, std::size_t) noexcept {
        std::free(p);
    }
    
    // 最大分配大小
    std::size_t max_size() const noexcept {
        return std::numeric_limits<std::size_t>::max() / sizeof(T);
    }
};

// 分配器比较
template<typename T, typename U>
bool operator==(const Allocator<T>&, const Allocator<U>&) noexcept {
    return true;
}

template<typename T, typename U>
bool operator!=(const Allocator<T>&, const Allocator<U>&) noexcept {
    return false;
}
```

### 3.2 内存池分配器

```cpp
// 内存池分配器实现
template<typename T, std::size_t BlockSize = 1024>
class PoolAllocator {
public:
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
    
    template<typename U>
    struct rebind {
        using other = PoolAllocator<U, BlockSize>;
    };
    
    PoolAllocator() noexcept
        : free_list_(nullptr) {}
    
    PoolAllocator(const PoolAllocator&) noexcept = default;
    
    template<typename U>
    PoolAllocator(const PoolAllocator<U, BlockSize>&) noexcept 
        : free_list_(nullptr) {}
    
    ~PoolAllocator() {
        clear();
    }
    
    T* allocate(std::size_t n) {
        if (n != 1) {
            throw std::bad_alloc();
        }
        
        if (free_list_) {
            T* result = static_cast<T*>(free_list_);
            free_list_ = free_list_->next;
            return result;
        }
        
        // 分配新块
        if (current_block_end_ - current_block_ptr_ < sizeof(T)) {
            allocate_new_block();
        }
        
        T* result = reinterpret_cast<T*>(current_block_ptr_);
        current_block_ptr_ += sizeof(T);
        return result;
    }
    
    void deallocate(T* p, std::size_t) noexcept {
        if (p) {
            auto node = static_cast<FreeListNode*>(p);
            node->next = free_list_;
            free_list_ = node;
        }
    }
    
private:
    struct FreeListNode {
        FreeListNode* next;
    };
    
    struct Block {
        char data[BlockSize];
        Block* next;
    };
    
    void allocate_new_block() {
        Block* new_block = new Block();
        new_block->next = blocks_;
        blocks_ = new_block;
        
        current_block_ptr_ = new_block->data;
        current_block_end_ = new_block->data + BlockSize;
    }
    
    void clear() {
        Block* block = blocks_;
        while (block) {
            Block* next = block->next;
            delete block;
            block = next;
        }
        blocks_ = nullptr;
        free_list_ = nullptr;
    }
    
    FreeListNode* free_list_;
    Block* blocks_ = nullptr;
    char* current_block_ptr_ = nullptr;
    char* current_block_end_ = nullptr;
};

template<typename T, std::size_t BS, typename U, std::size_t BS2>
bool operator==(const PoolAllocator<T, BS>&, 
                const PoolAllocator<U, BS2>&) noexcept {
    return BS == BS2;
}

template<typename T, std::size_t BS, typename U, std::size_t BS2>
bool operator!=(const PoolAllocator<T, BS>&, 
                const PoolAllocator<U, BS2>&) noexcept {
    return BS != BS2;
}
```

### 3.3 分配器面试问题

#### 面试问题5：为什么需要分配器？有什么优势？

**答案：**
分配器的优势：

1. **内存管理灵活性**：可以自定义内存分配策略
2. **性能优化**：内存池、对象池等技术
3. **特殊需求**：共享内存、内存映射等
4. **调试支持**：跟踪内存分配、检测泄漏

```cpp
// 跟踪分配器
template<typename T>
class TrackingAllocator {
public:
    using value_type = T;
    
    TrackingAllocator() {
        std::cout << "TrackingAllocator constructed" << std::endl;
    }
    
    template<typename U>
    TrackingAllocator(const TrackingAllocator<U>&) {}
    
    T* allocate(std::size_t n) {
        std::cout << "Allocating " << n << " objects" << std::endl;
        return std::allocator<T>().allocate(n);
    }
    
    void deallocate(T* p, std::size_t n) noexcept {
        std::cout << "Deallocating " << n << " objects" << std::endl;
        std::allocator<T>().deallocate(p, n);
    }
};

// 使用
std::vector<int, TrackingAllocator<int>> vec;
vec.push_back(42);
```

---

## 四、内存池技术

### 4.1 对象池实现

```cpp
// 对象池模板
template<typename T>
class ObjectPool {
public:
    ObjectPool(std::size_t initial_size = 10) {
        for (std::size_t i = 0; i < initial_size; ++i) {
            free_list_.push(new Object());
        }
    }
    
    ~ObjectPool() {
        while (!free_list_.empty()) {
            delete free_list_.front();
            free_list_.pop();
        }
    }
    
    template<typename... Args>
    T* acquire(Args&&... args) {
        if (free_list_.empty()) {
            free_list_.push(new Object());
        }
        
        Object* obj = free_list_.front();
        free_list_.pop();
        
        // 重用对象前调用析构函数
        obj->data.~T();
        
        // 在原位构造新对象
        new (&obj->data) T(std::forward<Args>(args)...);
        
        return &obj->data;
    }
    
    void release(T* obj) {
        obj->~T();
        free_list_.push(reinterpret_cast<Object*>(obj));
    }
    
private:
    struct Object {
        T data;
        Object* next;
    };
    
    std::queue<Object*> free_list_;
};

// 使用示例
class MyClass {
public:
    MyClass(int x, double y) : x_(x), y_(y) {
        std::cout << "MyClass constructed: " << x_ << ", " << y_ << std::endl;
    }
    
    ~MyClass() {
        std::cout << "MyClass destroyed" << std::endl;
    }
    
private:
    int x_;
    double y_;
};

void test_object_pool() {
    ObjectPool<MyClass> pool;
    
    auto obj1 = pool.acquire(10, 3.14);
    auto obj2 = pool.acquire(20, 6.28);
    
    pool.release(obj1);
    pool.release(obj2);
}
```

---

## 五、迭代器失效与稳定性

### 5.1 迭代器失效的底层原理

```cpp
// 迭代器失效分析
void iterator_invalidation_demo() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    auto it = vec.begin() + 2;  // 指向3
    
    // 插入元素可能导致重新分配
    vec.push_back(6);
    
    // it现在可能失效！
    // *it = 100;  // 未定义行为
    
    // 正确做法：在修改后重新获取迭代器
    it = vec.begin() + 2;
    *it = 100;  // 安全
}

// 各种容器的迭代器失效规则
/*
vector/deque/string:
- 插入：所有迭代器失效（可能重新分配）
- 删除：被删除元素及之后的所有迭代器失效

list/forward_list:
- 插入：所有迭代器保持有效
- 删除：只有被删除元素的迭代器失效

set/map/multiset/multimap:
- 插入：所有迭代器保持有效
- 删除：只有被删除元素的迭代器失效

unordered_set/unordered_map:
- 插入：所有迭代器可能失效（rehash）
- 删除：只有被删除元素的迭代器失效
*/
```

### 5.2 稳定迭代器技术

```cpp
// 使用索引而非迭代器
void safe_iteration(std::vector<int>& vec) {
    for (std::size_t i = 0; i < vec.size(); ++i) {
        if (vec[i] == 42) {
            vec.erase(vec.begin() + i);
            --i;  // 调整索引
        }
    }
}

// 使用erase-remove惯用法
void erase_remove_example(std::vector<int>& vec) {
    vec.erase(
        std::remove(vec.begin(), vec.end(), 42),
        vec.end()
    );
}
```

---

## 总结

本文档深入讲解了STL中的内存管理与RAII技术，包括：

1. **RAII原理与实现**：资源自动管理的核心技术
2. **智能指针技术**：unique_ptr、shared_ptr、weak_ptr的完整实现
3. **分配器技术**：自定义内存分配策略
4. **内存池技术**：高性能内存管理
5. **迭代器失效**：容器迭代器的安全性分析

这些技术是现代C++编程的基础，掌握它们对深入理解STL和编写高质量代码至关重要。