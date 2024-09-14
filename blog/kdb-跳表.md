点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. 跳表的定义
跳表（SkipList）是一种用于维护有序元素的链表结构，通过在链表基础上增加多层索引，实现快速查找和插入操作。跳表的层数是随机生成的，最上层节点较少，底层节点最多。  
每个节点拥有多个指向下一个节点的指针，不同的层级允许快速跳过多个节点。  
通过多层索引，跳表的查找和插入复杂度可以达到 O(logn)，接近二分查找的性能。  
kdb的跳表在[skiplist.h](https://github.com/2549141519/kdb/blob/main/src/include/skiplist.h)中实现。  

## 1.2 模板类定义
```
template<typename Key, class Comparator>
class SkipList {
```  
`Key`:跳表存储的键类型，可以是任何类型。  
`Comparator`:比较器，用于比较`Key`类型的值。跳表是有序的，因此需要一个比较器来确定键的顺序。  

## 1.3 内部节点结构`Node`
```
struct Node {
  Key const key;
  std::atomic<Node*> next_[1];  
};
```  
每个节点存储一个键值 `key` 和一个 `next_` 指针数组，`next_ `的大小根据节点的高度动态分配。每个指针指向跳表中不同层的下一个节点，跳表的结构通过这些指针数组实现多层索引。  
使用`atomic`类型确保无锁操作的线程安全性。  

# 2. 重要操作

## 2.1 插入操作`Insert`
```
void SkipList<Key, Comparator>::Insert(const Key& key) {
    Node* prev[KMaxHeight];
    Node* n = FindGreaterOrEqual(key, prev);  // 找到插入位置   
    assert(n == nullptr || !Equal(key, n->key));  // 跳表中不允许有重复键
    int height = RandomHeight();  // 随机生成节点的高度
    if (height > GetMaxHeight()) {
        for (int i = GetMaxHeight(); i < height; i++) {
            prev[i] = head_;
        }
        max_height_.store(height, std::memory_order_relaxed);
    }
    n = NewNode(key, height);  // 创建新节点
    for (int i = 0; i < height; i++) {
        n->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));  // 连接新节点
        prev[i]->SetNext(i, n);  // 设置前一个节点的下一指针
    }
}
```  
查找插入位置：  
调用 [`FindGreaterOrEqual`](#22-查找操作findgreaterorequal) 方法，找到跳表中第一个大于等于 `key` 的位置。`prev` 数组存储每一层插入位置前的节点。  

随机生成高度：  
通过 [`RandomHeight`](#23-随机高度生成randomheight) 随机生成节点的高度，这决定了新节点在跳表中有多少层索引。    

调整高度：  
如果新节点的高度超过当前最大高度，更新跳表的最大高度，并将新节点插入到相应的高度。  

连接节点：  
通过 `NoBarrier_SetNext` 和 `SetNext`，将新节点插入到跳表的每一层。  

## 2.2 查找操作`FindGreaterOrEqual`
```
Node* SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key, Node** prev) const {
    Node* n = head_;    
    int level = GetMaxHeight() - 1;
    while (true) {
        Node* next = n->Next(level);  // 获取当前层下一个节点
        if (KeyIsAfterNode(key, next)) {  // 如果下一个节点小于 key，继续往后找
            n = next;
        } else {
            if (prev != nullptr) prev[level] = n;  // 记录当前节点
            if (level == 0) return next;  // 找到第一个大于等于 key 的节点
            else level--;
        }
    }
}
```  
逐层查找：  
从最高层开始，逐层向下查找，直到找到第一个大于等于 `key` 的节点。  
优化查找：  
通过多层索引，可以快速跳过大量节点，减少查找时间。  
记录前一个节点：  
如果需要插入新节点，`prev` 数组会记录每层中当前节点的前驱节点。  

## 2.3 随机高度生成`RandomHeight`
```
int SkipList<Key, Comparator>::RandomHeight() {
    static const unsigned int kBranching = 4;
    int height = 1;
    while (height < KMaxHeight && rnd_.OneIn(kBranching)) {
        height++;
    }
    return height;
}
```   
随机生成高度：通过概率` 1/kBranching `来增加节点的高度。高度越高，节点越少，索引效率越高。  
最大高度限制：确保高度不超过 `KMaxHeight`（12层）。  

# 3. 线程安全性

## 3.1 无锁结构
`std::atomic<Node*> next_[1];`  
`std::atomic` 提供了一种可以在线程之间原子性地读写数据的机制，不需要锁。`next_` 是一个原子类型的指针数组，存储的是指向其他节点的指针。`std::atomic<Node*>` 确保在多个线程访问或修改 `next_` 指针时，能够避免竞争条件。  

## 3.2 内存序的使用

### 3.2.1 `memory_order_acquire`
`memory_order_acquire `用于保证对变量的读取操作不会在该读取之前的操作之后发生。即，任何读取操作都不会被重排序到 `memory_order_acquire `之前。  
```
Node* Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_acquire);
}
```  
`load(std::memory_order_acquire)`：在读取 `next_` 指针时，确保在 `next_` 被读取之后的操作不会被编译器或 CPU 重排序到读取操作之前。这意味着读取到的指针值是最新的，之前的所有写入操作已经对其他线程可见。  

### 3.2.2 `memory_order_release`
`memory_order_release `用于保证对变量的写入操作不会在该写入之后的操作之前发生。即，任何写入操作都不会被重排序到 `memory_order_release `之后。  
```
void SetNext(int n, Node *x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_release);
}
```  
`store(std::memory_order_release)`：在设置 `next_` 指针时，确保在写入新值之前的操作已经全部完成，并且对其他线程可见。这意味着在 `SetNext` 之前的任何内存操作都不会被重排序到这个存储操作之后。  

### 3.2.3 `memory_order_relaxed`
`memory_order_relaxed `是一种较弱的内存序，既不需要保证读取或写入的顺序，也不需要对其他线程可见。它仅确保操作本身是原子的，而不关心操作的顺序。  
```
Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
}

void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
}
```  
`memory_order_relaxed` 用于无需严格保证顺序的情况下，例如跳表中的某些内部操作。这些操作不会影响系统的全局状态，因此不需要严格的内存顺序保证。相较于 `acquire` 和 `release`，这种内存序可以进一步提高性能。  

# 4. 无锁跳表实现细节

## 4.1 查找操作 (`FindGreaterOrEqual`) 的无锁实现
在跳表中查找某个键时，不需要修改任何节点的结构，只需要读取节点的信息。因此，在查找操作[`FindGreaterOrEqual`](#22-查找操作findgreaterorequal)中使用了 `memory_order_acquire` 来读取节点指针。   
`Next(level) `使用 `memory_order_acquire` 确保读取到的节点指针是最新的，并且没有被其他线程重排序。  

## 4.2 插入操作 (`Insert`) 的无锁实现
插入操作[`Insert`](#21-插入操作insert)需要在多个层中更新节点指针，因此这里使用了 `memory_order_release` 来确保插入操作的顺序。  
`NoBarrier_SetNext` 使用 `memory_order_relaxed`，因为在构建新节点时不需要对指针设置严格的内存序。但在插入节点时，需要使用 `SetNext(i, n) `来确保在其他线程读取新节点时，插入顺序是正确的（通过 `memory_order_release` 保证）。  

# 5. 迭代器
跳表中提供了一个内部迭代器 `Iterator`，允许像标准库中的迭代器一样遍历跳表。  
```
class Iterator {
public:
    explicit Iterator(const SkipList* list);

    bool Valid() const;
    const Key& key() const;
    void Next();
    void Prev();
    void SeekToFirst();
    void SeekToLast();
    void Seek(const Key& target);
};
```   
`SeekToFirst`：将迭代器指向跳表的第一个节点。  
`SeekToLast`：将迭代器指向跳表的最后一个节点。  
`Seek`：将迭代器指向大于等于指定 key 的第一个节点。  