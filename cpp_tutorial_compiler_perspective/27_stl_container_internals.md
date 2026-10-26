# STL Container Internals

## Introduction

Understanding the internal implementation of STL containers is crucial for writing efficient C++ code. This knowledge helps developers make informed decisions about which container to use for specific scenarios, understand performance characteristics, and appreciate the design decisions that went into the standard library implementation.

## std::vector Implementation

### Basic Vector Structure

```cpp
// Internal representation of std::vector
template<typename T, typename Allocator = std::allocator<T>>
class VectorInternals {
private:
    T* data_;           // Pointer to the beginning of the storage
    std::size_t size_;  // Number of elements currently in the vector
    std::size_t capacity_; // Total number of elements that can be stored
    Allocator alloc_;   // Allocator instance
    
    // Growth strategy: typically 1.5x or 2x when resizing
    static constexpr double GROWTH_FACTOR = 1.5;
    
public:
    VectorInternals() : data_(nullptr), size_(0), capacity_(0) {}
    
    ~VectorInternals() {
        clear();
        if (data_) {
            std::allocator_traits<Allocator>::deallocate(alloc_, data_, capacity_);
        }
    }
    
    void push_back(const T& value) {
        if (size_ == capacity_) {
            reserve(capacity_ == 0 ? 1 : static_cast<std::size_t>(capacity_ * GROWTH_FACTOR));
        }
        
        std::allocator_traits<Allocator>::construct(alloc_, data_ + size_, value);
        ++size_;
    }
    
    void push_back(T&& value) {
        if (size_ == capacity_) {
            reserve(capacity_ == 0 ? 1 : static_cast<std::size_t>(capacity_ * GROWTH_FACTOR));
        }
        
        std::allocator_traits<Allocator>::construct(alloc_, data_ + size_, std::move(value));
        ++size_;
    }
    
    void reserve(std::size_t new_capacity) {
        if (new_capacity <= capacity_) return;
        
        T* new_data = std::allocator_traits<Allocator>::allocate(alloc_, new_capacity);
        
        // Move existing elements to new storage
        for (std::size_t i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::construct(alloc_, new_data + i, 
                                                      std::move(data_[i]));
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        }
        
        // Deallocate old storage
        std::allocator_traits<Allocator>::deallocate(alloc_, data_, capacity_);
        
        data_ = new_data;
        capacity_ = new_capacity;
    }
    
    void resize(std::size_t new_size) {
        if (new_size < size_) {
            // Destroy excess elements
            for (std::size_t i = new_size; i < size_; ++i) {
                std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
            }
        } else if (new_size > size_) {
            if (new_size > capacity_) {
                reserve(new_size);
            }
            // Construct new elements with default constructor
            for (std::size_t i = size_; i < new_size; ++i) {
                std::allocator_traits<Allocator>::construct(alloc_, data_ + i);
            }
        }
        size_ = new_size;
    }
    
    void clear() {
        for (std::size_t i = 0; i < size_; ++i) {
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        }
        size_ = 0;
    }
    
    T& operator[](std::size_t index) { return data_[index]; }
    const T& operator[](std::size_t index) const { return data_[index]; }
    std::size_t size() const { return size_; }
    std::size_t capacity() const { return capacity_; }
    bool empty() const { return size_ == 0; }
    
    T* data() { return data_; }
    const T* data() const { return data_; }
};
```

### Small Vector Optimization

```cpp
// Implementation of Small Vector Optimization (SVO)
template<typename T, std::size_t InlineCapacity = 8>
class SmallVector {
private:
    union {
        T* heap_ptr_;
        alignas(T) char inline_storage_[sizeof(T) * InlineCapacity];
    };
    
    std::size_t size_;
    std::size_t capacity_;
    bool using_heap_;
    
    T* data_ptr() {
        return using_heap_ ? heap_ptr_ : reinterpret_cast<T*>(inline_storage_);
    }
    
    const T* data_ptr() const {
        return using_heap_ ? heap_ptr_ : reinterpret_cast<const T*>(inline_storage_);
    }
    
    void allocate_heap(std::size_t new_capacity) {
        T* new_data = static_cast<T*>(std::malloc(new_capacity * sizeof(T)));
        if (!new_data) throw std::bad_alloc();
        
        if (using_heap_) {
            std::free(heap_ptr_);
        } else {
            // Move from inline storage to heap
            for (std::size_t i = 0; i < size_; ++i) {
                new(new_data + i) T(std::move(reinterpret_cast<T*>(inline_storage_)[i]));
                reinterpret_cast<T*>(inline_storage_)[i].~T();
            }
        }
        
        heap_ptr_ = new_data;
        capacity_ = new_capacity;
        using_heap_ = true;
    }
    
public:
    SmallVector() : size_(0), capacity_(InlineCapacity), using_heap_(false) {}
    
    ~SmallVector() {
        for (std::size_t i = 0; i < size_; ++i) {
            data_ptr()[i].~T();
        }
        
        if (using_heap_) {
            std::free(heap_ptr_);
        }
    }
    
    void push_back(const T& value) {
        if (size_ == capacity_) {
            std::size_t new_capacity = capacity_ * 2;
            allocate_heap(new_capacity);
        }
        
        new(data_ptr() + size_) T(value);
        ++size_;
    }
    
    std::size_t size() const { return size_; }
    std::size_t capacity() const { return capacity_; }
    bool using_inline_storage() const { return !using_heap_; }
};
```

## std::list and std::forward_list

### Doubly Linked List Implementation

```cpp
// Internal structure of std::list
template<typename T, typename Allocator = std::allocator<T>>
class ListInternals {
private:
    struct Node {
        T data;
        Node* next;
        Node* prev;
        
        Node(const T& value) : data(value), next(nullptr), prev(nullptr) {}
        Node(T&& value) : data(std::move(value)), next(nullptr), prev(nullptr) {}
    };
    
    using node_allocator_type = typename std::allocator_traits<Allocator>::
                               template rebind_alloc<Node>;
    node_allocator_type node_alloc_;
    
    Node* head_;
    Node* tail_;
    std::size_t size_;
    
    Node* create_node(const T& value) {
        Node* node = std::allocator_traits<node_allocator_type>::allocate(node_alloc_, 1);
        std::allocator_traits<node_allocator_type>::construct(node_alloc_, node, value);
        return node;
    }
    
    Node* create_node(T&& value) {
        Node* node = std::allocator_traits<node_allocator_type>::allocate(node_alloc_, 1);
        std::allocator_traits<node_allocator_type>::construct(node_alloc_, node, std::move(value));
        return node;
    }
    
    void destroy_node(Node* node) {
        std::allocator_traits<node_allocator_type>::destroy(node_alloc_, node);
        std::allocator_traits<node_allocator_type>::deallocate(node_alloc_, node, 1);
    }
    
public:
    ListInternals() : head_(nullptr), tail_(nullptr), size_(0) {
        // Create sentinel node for easier implementation
        Node* sentinel = create_node(T{});
        head_ = sentinel;
        tail_ = sentinel;
        sentinel->next = sentinel;
        sentinel->prev = sentinel;
    }
    
    ~ListInternals() {
        clear();
        destroy_node(head_); // The sentinel node
    }
    
    void push_front(const T& value) {
        Node* new_node = create_node(value);
        insert_after(head_, new_node);
    }
    
    void push_back(const T& value) {
        Node* new_node = create_node(value);
        insert_before(tail_, new_node);
    }
    
    void insert_after(Node* pos, Node* new_node) {
        new_node->next = pos->next;
        new_node->prev = pos;
        pos->next->prev = new_node;
        pos->next = new_node;
        ++size_;
    }
    
    void insert_before(Node* pos, Node* new_node) {
        new_node->prev = pos->prev;
        new_node->next = pos;
        pos->prev->next = new_node;
        pos->prev = new_node;
        ++size_;
    }
    
    void erase(Node* node) {
        if (node == head_) return; // Don't erase sentinel
        
        node->prev->next = node->next;
        node->next->prev = node->prev;
        destroy_node(node);
        --size_;
    }
    
    void clear() {
        Node* current = head_->next;
        while (current != head_) {
            Node* next = current->next;
            destroy_node(current);
            current = next;
        }
        head_->next = head_;
        head_->prev = head_;
        size_ = 0;
    }
    
    std::size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
    
    Node* begin() { return head_->next; }
    Node* end() { return head_; }
    const Node* begin() const { return head_->next; }
    const Node* end() const { return head_; }
};
```

## std::deque Implementation

### Deque Internal Structure

```cpp
// Internal structure of std::deque
template<typename T, typename Allocator = std::allocator<T>>
class DequeInternals {
private:
    static constexpr std::size_t BUFFER_SIZE = 
        (sizeof(T) < 256) ? 512 / sizeof(T) : 1;
    
    struct MapNode {
        T* buffer;
    };
    
    MapNode* map_;
    std::size_t map_size_;
    T** start_buffer_;
    T** finish_buffer_;
    T* start_pos_;
    T* finish_pos_;
    Allocator alloc_;
    
    void create_buffer(T** buffer_ptr) {
        *buffer_ptr = std::allocator_traits<Allocator>::allocate(alloc_, BUFFER_SIZE);
    }
    
    void destroy_buffer(T** buffer_ptr) {
        if (*buffer_ptr) {
            std::allocator_traits<Allocator>::deallocate(alloc_, *buffer_ptr, BUFFER_SIZE);
            *buffer_ptr = nullptr;
        }
    }
    
    std::size_t buffer_index(const T* ptr) const {
        return (ptr - *start_buffer_) / BUFFER_SIZE;
    }
    
public:
    DequeInternals() : map_(nullptr), map_size_(0), 
                      start_buffer_(nullptr), finish_buffer_(nullptr),
                      start_pos_(nullptr), finish_pos_(nullptr) {
        initialize();
    }
    
    ~DequeInternals() {
        clear();
        for (T** buf = start_buffer_; buf <= finish_buffer_; ++buf) {
            destroy_buffer(buf);
        }
        if (map_) {
            std::allocator_traits<Allocator>::deallocate(alloc_, 
                                                       reinterpret_cast<MapNode*>(map_), 
                                                       map_size_);
        }
    }
    
    void push_back(const T& value) {
        if (finish_pos_ != (*finish_buffer_ + BUFFER_SIZE - 1)) {
            // There's space in the current buffer
            std::allocator_traits<Allocator>::construct(alloc_, finish_pos_, value);
            ++finish_pos_;
        } else {
            // Need a new buffer
            push_back_aux(value);
        }
    }
    
    void push_front(const T& value) {
        if (start_pos_ != *start_buffer_) {
            // There's space in the current buffer
            --start_pos_;
            std::allocator_traits<Allocator>::construct(alloc_, start_pos_, value);
        } else {
            // Need a new buffer
            push_front_aux(value);
        }
    }
    
private:
    void initialize() {
        map_size_ = 8; // Initial map size
        map_ = reinterpret_cast<T**>(std::allocator_traits<Allocator>::
                                   allocate(alloc_, map_size_ * sizeof(T*)));
        
        T** center = map_ + map_size_ / 2;
        start_buffer_ = center;
        finish_buffer_ = center;
        create_buffer(start_buffer_);
        
        start_pos_ = *start_buffer_ + BUFFER_SIZE / 2;
        finish_pos_ = start_pos_;
    }
    
    void push_back_aux(const T& value) {
        // Create new buffer if needed
        if (finish_buffer_ == map_ + map_size_ - 1) {
            reallocate_map(true);
        }
        
        ++finish_buffer_;
        create_buffer(finish_buffer_);
        finish_pos_ = *finish_buffer_;
        
        std::allocator_traits<Allocator>::construct(alloc_, finish_pos_, value);
        ++finish_pos_;
    }
    
    void push_front_aux(const T& value) {
        // Create new buffer if needed
        if (start_buffer_ == map_) {
            reallocate_map(false);
        }
        
        --start_buffer_;
        create_buffer(start_buffer_);
        start_pos_ = *start_buffer_ + BUFFER_SIZE - 1;
        
        std::allocator_traits<Allocator>::construct(alloc_, start_pos_, value);
    }
    
    void reallocate_map(bool add_at_back) {
        std::size_t old_size = finish_buffer_ - start_buffer_ + 1;
        std::size_t new_size = old_size * 2;
        
        T** new_map = reinterpret_cast<T**>(std::allocator_traits<Allocator>::
                                          allocate(alloc_, new_size * sizeof(T*)));
        
        T** new_start = new_map + (new_size - old_size) / 2;
        T** new_finish = new_start + old_size - 1;
        
        // Copy buffer pointers
        std::copy(start_buffer_, finish_buffer_ + 1, new_start);
        
        // Update pointers
        start_buffer_ = new_start;
        finish_buffer_ = new_finish;
        
        // Deallocate old map
        std::allocator_traits<Allocator>::deallocate(alloc_, 
                                                   reinterpret_cast<MapNode*>(map_), 
                                                   map_size_);
        map_ = new_map;
        map_size_ = new_size;
    }
    
public:
    std::size_t size() const {
        if (finish_buffer_ == start_buffer_) {
            return finish_pos_ - start_pos_;
        }
        return (finish_buffer_ - start_buffer_) * BUFFER_SIZE +
               (finish_pos_ - *finish_buffer_) +
               (*start_buffer_ - start_pos_);
    }
    
    bool empty() const { return size() == 0; }
    
    T& operator[](std::size_t n) {
        T** buffer = start_buffer_ + n / BUFFER_SIZE;
        return (*buffer)[n % BUFFER_SIZE];
    }
    
    const T& operator[](std::size_t n) const {
        T** buffer = start_buffer_ + n / BUFFER_SIZE;
        return (*buffer)[n % BUFFER_SIZE];
    }
};
```

## Associative Containers

### Red-Black Tree Implementation

```cpp
// Internal structure for std::set and std::map
template<typename Key, typename Compare = std::less<Key>>
class RBTreeInternals {
protected:
    enum Color { RED, BLACK };
    
    struct Node {
        Key key;
        Color color;
        Node* parent;
        Node* left;
        Node* right;
        
        Node(const Key& k) : key(k), color(RED), parent(nullptr), left(nullptr), right(nullptr) {}
        Node(Key&& k) : key(std::move(k)), color(RED), parent(nullptr), left(nullptr), right(nullptr) {}
    };
    
    Node* root_;
    Node* nil_;  // Sentinel node representing null
    Compare comp_;
    std::size_t size_;
    
    void left_rotate(Node* x) {
        Node* y = x->right;
        x->right = y->left;
        
        if (y->left != nil_) {
            y->left->parent = x;
        }
        
        y->parent = x->parent;
        
        if (x->parent == nil_) {
            root_ = y;
        } else if (x == x->parent->left) {
            x->parent->left = y;
        } else {
            x->parent->right = y;
        }
        
        y->left = x;
        x->parent = y;
    }
    
    void right_rotate(Node* y) {
        Node* x = y->left;
        y->left = x->right;
        
        if (x->right != nil_) {
            x->right->parent = y;
        }
        
        x->parent = y->parent;
        
        if (y->parent == nil_) {
            root_ = x;
        } else if (y == y->parent->right) {
            y->parent->right = x;
        } else {
            y->parent->left = x;
        }
        
        x->right = y;
        y->parent = x;
    }
    
    void insert_fixup(Node* z) {
        while (z->parent->color == RED) {
            if (z->parent == z->parent->parent->left) {
                Node* y = z->parent->parent->right;
                if (y->color == RED) {
                    z->parent->color = BLACK;
                    y->color = BLACK;
                    z->parent->parent->color = RED;
                    z = z->parent->parent;
                } else {
                    if (z == z->parent->right) {
                        z = z->parent;
                        left_rotate(z);
                    }
                    z->parent->color = BLACK;
                    z->parent->parent->color = RED;
                    right_rotate(z->parent->parent);
                }
            } else {
                Node* y = z->parent->parent->left;
                if (y->color == RED) {
                    z->parent->color = BLACK;
                    y->color = BLACK;
                    z->parent->parent->color = RED;
                    z = z->parent->parent;
                } else {
                    if (z == z->parent->left) {
                        z = z->parent;
                        right_rotate(z);
                    }
                    z->parent->color = BLACK;
                    z->parent->parent->color = RED;
                    left_rotate(z->parent->parent);
                }
            }
        }
        root_->color = BLACK;
    }
    
    Node* minimum(Node* node) const {
        while (node->left != nil_) {
            node = node->left;
        }
        return node;
    }
    
    Node* maximum(Node* node) const {
        while (node->right != nil_) {
            node = node->right;
        }
        return node;
    }
    
    Node* successor(Node* x) const {
        if (x->right != nil_) {
            return minimum(x->right);
        }
        Node* y = x->parent;
        while (y != nil_ && x == y->right) {
            x = y;
            y = y->parent;
        }
        return y;
    }
    
    void transplant(Node* u, Node* v) {
        if (u->parent == nil_) {
            root_ = v;
        } else if (u == u->parent->left) {
            u->parent->left = v;
        } else {
            u->parent->right = v;
        }
        v->parent = u->parent;
    }
    
    void delete_fixup(Node* x) {
        while (x != root_ && x->color == BLACK) {
            if (x == x->parent->left) {
                Node* w = x->parent->right;
                if (w->color == RED) {
                    w->color = BLACK;
                    x->parent->color = RED;
                    left_rotate(x->parent);
                    w = x->parent->right;
                }
                if (w->left->color == BLACK && w->right->color == BLACK) {
                    w->color = RED;
                    x = x->parent;
                } else {
                    if (w->right->color == BLACK) {
                        w->left->color = BLACK;
                        w->color = RED;
                        right_rotate(w);
                        w = x->parent->right;
                    }
                    w->color = x->parent->color;
                    x->parent->color = BLACK;
                    w->right->color = BLACK;
                    left_rotate(x->parent);
                    x = root_;
                }
            } else {
                Node* w = x->parent->left;
                if (w->color == RED) {
                    w->color = BLACK;
                    x->parent->color = RED;
                    right_rotate(x->parent);
                    w = x->parent->left;
                }
                if (w->right->color == BLACK && w->left->color == BLACK) {
                    w->color = RED;
                    x = x->parent;
                } else {
                    if (w->left->color == BLACK) {
                        w->right->color = BLACK;
                        w->color = RED;
                        left_rotate(w);
                        w = x->parent->left;
                    }
                    w->color = x->parent->color;
                    x->parent->color = BLACK;
                    w->left->color = BLACK;
                    right_rotate(x->parent);
                    x = root_;
                }
            }
        }
        x->color = BLACK;
    }
    
public:
    RBTreeInternals() : size_(0) {
        nil_ = new Node(Key{});
        nil_->color = BLACK;
        root_ = nil_;
    }
    
    ~RBTreeInternals() {
        clear();
        delete nil_;
    }
    
    Node* insert(const Key& key) {
        Node* z = new Node(key);
        Node* y = nil_;
        Node* x = root_;
        
        while (x != nil_) {
            y = x;
            if (comp_(z->key, x->key)) {
                x = x->left;
            } else if (comp_(x->key, z->key)) {
                x = x->right;
            } else {
                // Key already exists, don't insert
                delete z;
                return x; // Return existing node
            }
        }
        
        z->parent = y;
        if (y == nil_) {
            root_ = z;
        } else if (comp_(z->key, y->key)) {
            y->left = z;
        } else {
            y->right = z;
        }
        
        z->left = nil_;
        z->right = nil_;
        z->color = RED;
        
        insert_fixup(z);
        ++size_;
        return z;
    }
    
    void remove(Node* z) {
        Node* y = z;
        Color y_original_color = y->color;
        Node* x;
        
        if (z->left == nil_) {
            x = z->right;
            transplant(z, z->right);
        } else if (z->right == nil_) {
            x = z->left;
            transplant(z, z->left);
        } else {
            y = minimum(z->right);
            y_original_color = y->color;
            x = y->right;
            
            if (y->parent == z) {
                x->parent = y;
            } else {
                transplant(y, y->right);
                y->right = z->right;
                y->right->parent = y;
            }
            
            transplant(z, y);
            y->left = z->left;
            y->left->parent = y;
            y->color = z->color;
        }
        
        if (y_original_color == BLACK) {
            delete_fixup(x);
        }
        
        delete z;
        --size_;
    }
    
    Node* find(const Key& key) const {
        Node* current = root_;
        while (current != nil_) {
            if (comp_(key, current->key)) {
                current = current->left;
            } else if (comp_(current->key, key)) {
                current = current->right;
            } else {
                return current; // Found
            }
        }
        return nil_; // Not found
    }
    
    void clear() {
        clear_subtree(root_);
        root_ = nil_;
        size_ = 0;
    }
    
private:
    void clear_subtree(Node* node) {
        if (node != nil_) {
            clear_subtree(node->left);
            clear_subtree(node->right);
            delete node;
        }
    }
    
public:
    std::size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
};
```

### Hash Table Implementation (std::unordered_map/set)

```cpp
// Internal structure of hash containers
template<typename Key, typename Hash = std::hash<Key>, typename Equal = std::equal_to<Key>>
class HashTableInternals {
private:
    struct Node {
        Key key;
        Node* next;
        
        Node(const Key& k) : key(k), next(nullptr) {}
        Node(Key&& k) : key(std::move(k)), next(nullptr) {}
    };
    
    std::vector<Node*> buckets_;
    std::size_t size_;
    std::size_t bucket_count_;
    double max_load_factor_;
    Hash hasher_;
    Equal equal_;
    
    std::size_t hash_to_bucket(const Key& key) const {
        return hasher_(key) % bucket_count_;
    }
    
    void rehash_if_needed() {
        if (static_cast<double>(size_) / bucket_count_ > max_load_factor_) {
            rehash(bucket_count_ * 2);
        }
    }
    
    void rehash(std::size_t new_bucket_count) {
        std::vector<Node*> old_buckets = std::move(buckets_);
        bucket_count_ = new_bucket_count;
        buckets_.assign(bucket_count_, nullptr);
        size_ = 0;
        
        for (Node* head : old_buckets) {
            while (head) {
                Node* next = head->next;
                
                std::size_t new_bucket = hash_to_bucket(head->key);
                head->next = buckets_[new_bucket];
                buckets_[new_bucket] = head;
                
                head = next;
                ++size_;
            }
        }
    }
    
public:
    HashTableInternals(std::size_t initial_buckets = 16) 
        : bucket_count_(initial_buckets), size_(0), max_load_factor_(1.0) {
        buckets_.assign(bucket_count_, nullptr);
    }
    
    ~HashTableInternals() {
        clear();
    }
    
    Node* insert(const Key& key) {
        std::size_t bucket = hash_to_bucket(key);
        
        // Check if key already exists
        for (Node* current = buckets_[bucket]; current; current = current->next) {
            if (equal_(current->key, key)) {
                return current; // Key already exists
            }
        }
        
        // Create new node
        Node* new_node = new Node(key);
        new_node->next = buckets_[bucket];
        buckets_[bucket] = new_node;
        ++size_;
        
        rehash_if_needed();
        return new_node;
    }
    
    Node* insert(Key&& key) {
        std::size_t bucket = hash_to_bucket(key);
        
        // Check if key already exists
        for (Node* current = buckets_[bucket]; current; current = current->next) {
            if (equal_(current->key, key)) {
                return current; // Key already exists
            }
        }
        
        // Create new node
        Node* new_node = new Node(std::move(key));
        new_node->next = buckets_[bucket];
        buckets_[bucket] = new_node;
        ++size_;
        
        rehash_if_needed();
        return new_node;
    }
    
    void remove(const Key& key) {
        std::size_t bucket = hash_to_bucket(key);
        Node* current = buckets_[bucket];
        Node* prev = nullptr;
        
        while (current) {
            if (equal_(current->key, key)) {
                if (prev) {
                    prev->next = current->next;
                } else {
                    buckets_[bucket] = current->next;
                }
                delete current;
                --size_;
                return;
            }
            prev = current;
            current = current->next;
        }
    }
    
    Node* find(const Key& key) const {
        std::size_t bucket = hash_to_bucket(key);
        for (Node* current = buckets_[bucket]; current; current = current->next) {
            if (equal_(current->key, key)) {
                return current;
            }
        }
        return nullptr;
    }
    
    void clear() {
        for (Node*& head : buckets_) {
            while (head) {
                Node* temp = head;
                head = head->next;
                delete temp;
            }
        }
        size_ = 0;
    }
    
    std::size_t size() const { return size_; }
    std::size_t bucket_count() const { return bucket_count_; }
    bool empty() const { return size_ == 0; }
    double load_factor() const { return static_cast<double>(size_) / bucket_count_; }
};
```

## Performance Characteristics

### Big O Notations and Complexity

```cpp
// Performance characteristics of different containers
class ContainerPerformance {
public:
    void complexity_summary() {
        std::cout << "std::vector:" << std::endl;
        std::cout << "  - Random access: O(1)" << std::endl;
        std::cout << "  - Insert at end: O(1) amortized, O(n) worst case" << std::endl;
        std::cout << "  - Insert/delete in middle: O(n)" << std::endl;
        std::cout << "  - Memory: Contiguous, cache-friendly" << std::endl;
        
        std::cout << "\nstd::list:" << std::endl;
        std::cout << "  - Random access: O(n)" << std::endl;
        std::cout << "  - Insert/delete at known position: O(1)" << std::endl;
        std::cout << "  - Memory: Non-contiguous, extra overhead per element" << std::endl;
        
        std::cout << "\nstd::deque:" << std::endl;
        std::cout << "  - Random access: O(1)" << std::endl;
        std::cout << "  - Insert/delete at front/back: O(1) amortized" << std::endl;
        std::cout << "  - Insert/delete in middle: O(n)" << std::endl;
        std::cout << "  - Memory: Multiple contiguous blocks" << std::endl;
        
        std::cout << "\nstd::map/std::set:" << std::endl;
        std::cout << "  - Search/insert/delete: O(log n)" << std::endl;
        std::cout << "  - Memory: Sorted, tree-based structure" << std::endl;
        
        std::cout << "\nstd::unordered_map/std::unordered_set:" << std::endl;
        std::cout << "  - Search/insert/delete: O(1) average, O(n) worst case" << std::endl;
        std::cout << "  - Memory: Hash table with potential for collisions" << std::endl;
    }
    
    // Benchmark different containers
    template<typename Container>
    double benchmark_insertion(std::size_t n) {
        Container c;
        auto start = std::chrono::high_resolution_clock::now();
        
        for (std::size_t i = 0; i < n; ++i) {
            c.insert(c.end(), i);
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration<double, std::milli>(end - start).count();
    }
    
    void performance_comparison() {
        const std::size_t size = 100000;
        
        std::cout << "Performance comparison for " << size << " elements:" << std::endl;
        
        std::cout << "std::vector: " << benchmark_insertion<std::vector<int>>(size) << " ms" << std::endl;
        std::cout << "std::list: " << benchmark_insertion<std::list<int>>(size) << " ms" << std::endl;
        std::cout << "std::deque: " << benchmark_insertion<std::deque<int>>(size) << " ms" << std::endl;
    }
};
```

## Memory Layout and Cache Considerations

### Cache Performance Analysis

```cpp
// Understanding cache performance in STL containers
class CachePerformanceAnalysis {
public:
    // Cache-friendly iteration patterns
    void cache_friendly_iteration() {
        std::vector<int> vec(1000000);
        std::iota(vec.begin(), vec.end(), 1);
        
        // Cache-friendly: sequential access
        volatile long sum = 0;  // volatile to prevent optimization
        for (std::size_t i = 0; i < vec.size(); ++i) {
            sum += vec[i];
        }
        
        // Cache-unfriendly: strided access
        volatile long strided_sum = 0;
        for (std::size_t i = 0; i < vec.size(); i += 16) {  // 16 = typical cache line size in ints
            strided_sum += vec[i];
        }
    }
    
    // Memory layout comparison
    void memory_layout_comparison() {
        std::cout << "Memory layout comparison:" << std::endl;
        
        std::vector<int> vec = {1, 2, 3, 4, 5};
        std::list<int> lst = {1, 2, 3, 4, 5};
        std::deque<int> deq = {1, 2, 3, 4, 5};
        
        std::cout << "std::vector<int>: Contiguous memory" << std::endl;
        std::cout << "  Address of elements: ";
        for (const auto& x : vec) {
            std::cout << "&" << x << " ";
        }
        std::cout << std::endl;
        
        std::cout << "std::list<int>: Dispersed memory" << std::endl;
        std::cout << "std::deque<int>: Multiple contiguous blocks" << std::endl;
    }
    
    // Performance impact of different container choices
    template<typename Container>
    double measure_iteration_performance(Container& container) {
        auto start = std::chrono::high_resolution_clock::now();
        
        volatile typename Container::value_type sum = 0;
        for (const auto& item : container) {
            sum += item;
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration<double, std::milli>(end - start).count();
    }
    
    void container_performance_test() {
        const std::size_t size = 1000000;
        
        std::vector<int> vec(size);
        std::iota(vec.begin(), vec.end(), 1);
        
        std::list<int> lst(size);
        std::iota(lst.begin(), lst.end(), 1);
        
        std::deque<int> deq(size);
        std::iota(deq.begin(), deq.end(), 1);
        
        std::cout << "Iteration performance comparison:" << std::endl;
        std::cout << "std::vector: " << measure_iteration_performance(vec) << " ms" << std::endl;
        std::cout << "std::list: " << measure_iteration_performance(lst) << " ms" << std::endl;
        std::cout << "std::deque: " << measure_iteration_performance(deq) << " ms" << std::endl;
    }
};
```

Understanding STL container internals is essential for making informed decisions about which container to use for specific scenarios, considering factors like performance, memory usage, and the nature of operations you'll be performing.