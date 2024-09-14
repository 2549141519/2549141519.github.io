点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. 代码实现
kdb的内存池由[arena.h](https://github.com/2549141519/kdb/blob/main/src/utils/arena.h)和[arena.cpp](https://github.com/2549141519/kdb/blob/main/src/utils/arena.cpp)实现。  

实现内存池能够有效减少内存分配和释放的开销，提高系统性能。内存池的基本思想是预先分配一块大内存，并根据需要从中分配小块给应用程序使用，而不是每次都通过系统调用去动态分配和释放内存。  

代码实现了一个用于高效内存分配的类`Arena`，通过预分配一块较大的内存块，并在其上进行内存分配。  

# 2. 代码解析

## 2.1 类的基本构造
`blocks_`: 保存所有分配的内存块，每一个内存块的大小由 `kBlockSize` 决定。所有分配的内存块的指针存储在一个 `std::vector<char*>` 中。  

`alloc_ptr_`: 当前分配指针，指向内存池中下一个可用的内存位置。  

`alloc_bytes_remaining_`: 当前块剩余的内存大小，用来记录当前内存块中还可以分配的字节数。  

`memory_usage_`: 用 `std::atomic<std::size_t>` 类型记录总的内存使用量，适合多线程环境。  

## 2.2 构造和析构
构造函数初始化了 `alloc_ptr_`（当前指针）为 `nullptr`，并将剩余内存字节数设为 0，表示当前没有可以分配的内存块。  

析构函数通过遍历 `blocks_`，逐一释放已分配的内存块，避免内存泄漏。  

## 2.3 内存分配
`Allocate`: 这是对外暴露的内存分配接口。如果当前内存块中剩余的空间足够满足需求，则直接分配，否则调用 `AllocateFallback`，即分配新的内存块。  

`AllocateFallback`: [AllocateFallback](#26-allocatefallback中的块大小策略)在当前块的剩余内存不足时调用。首先检查所需的字节数是否超过 `kBlockSize / 4`（即 1024 字节）。如果超出，则直接为其分配一个新的块，大小为所需字节数；否则，分配一个常规的块，大小为 `kBlockSize`（4096 字节），并从其中分配所需的内存。  

`AllocateNewBlock`: 该函数用于分配一个新的内存块。分配完新块后，存储块指针，并更新总的内存使用量。注意这里内存使用量增加的值是新块大小加上存储块指针的大小。  

## 2.4 内存对齐
`AllocateAligned`

## 2.5 多线程支持
`memory_usage_ `使用了 `std::atomic` 来保证在多线程环境下的安全访问。这意味着多个线程可以同时分配内存。  

但在实现时，默认假设了`Arena`主要用于单线程环境，因此其他部分（如`alloc_ptr_ `和`alloc_bytes_remaining_ `）没有使用锁机制。  

## 2.6 AllocateFallback中的块大小策略
```
char *Arena::AllocateFallback(std::size_t bytes)
    {
        if (bytes > kBlockSize / 4) 
        {
            char* result = AllocateNewBlock(bytes);
            return result;
        }
        alloc_ptr_ = AllocateNewBlock(kBlockSize);
        alloc_bytes_remaining_ = kBlockSize;

        char* result = alloc_ptr_;
        alloc_ptr_ += bytes;
        alloc_bytes_remaining_ -= bytes;
        return result;
    }
```  

### 2.6.1 内存块小于或等于 `kBlockSize / 4 `的情况
对于小于等于 1024 字节的分配请求，`Arena `会预先分配一个较大的内存块（大小为 4096 字节），并将多次分配集中在同一块内存中。  

这种策略的优点是可以有效减少频繁调用` malloc `或` new `等系统内存分配函数的开销，因为系统级别的内存分配操作相对较慢。
每个块都是 4096 字节，确保了即使是多个小内存块的分配，也可以有效利用这块内存。  

避免内存碎片:   
小块的内存分配会从当前块中继续分配，直到该块的剩余空间不足。这种策略减少了内存碎片，并且较好地利用了已经分配的内存。  

缓存友好性:   
通过在同一块大内存中分配多个小块，可以提高数据在 CPU 缓存中的局部性，有助于提高内存访问性能。  

### 2.6.2 内存块大于 `kBlockSize / 4` 的情况
当请求的内存大于 1024 字节时，`Arena` 会直接为这个请求分配一块大小等于 `bytes` 的内存，而不是从一个 4096 字节的块中分配。这意味着大块的内存请求会得到独立的一整块内存。  

这种策略的优点在于避免了较大内存块分配后，导致 4096 字节内存块中剩余的空间无法有效利用的情况。假如请求 3000 字节，如果仍然从 4096 字节的块中分配，剩下的 1096 字节可能很难再用来满足其他请求，造成浪费。  

通过为较大请求分配独立的内存块，可以确保内存的分配效率。  

减少大内存请求对整体内存池的影响:   
当大于 1024 字节的请求需要独立的内存块时，它不会影响到已经存在的小块分配，也不会浪费内存池中较大的块。这样可以将小内存分配和大内存分配的需求分开处理，避免相互干扰。  