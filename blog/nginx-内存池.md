点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. 介绍
Nginx 内存池的实现是其高性能、高并发处理能力的重要组成部分之一。Nginx 的内存池主要用于高效管理内存，避免频繁的内存分配与释放操作，减少内存碎片，提升性能。Nginx 内存池的实现遵循“预分配一块内存，然后进行小块分配”的策略，类似于常见的内存池模型。  
Nginx在 [ngx_palloc.h](https://github.com/nginx/nginx/blob/master/src/core/ngx_palloc.h) 和 [ngx_palloc.c](https://github.com/nginx/nginx/blob/master/src/core/ngx_palloc.c) 中实现了内存池。  

# 2. 内存池的实现原理

## 2.1 Nginx 内存池的结构
`ngx_pool_s` 结构体是内存池的核心结构，它管理内存块的链表、内存池的大小信息、以及清理函数等。其定义如下：  
```
struct ngx_pool_s {
    ngx_pool_data_t       d;         // 内存块的数据
    size_t                max;       // 可从该内存池分配的最大内存块大小
    ngx_pool_t           *current;   // 当前活跃的内存块
    ngx_chain_t          *chain;     // 缓存链
    ngx_pool_large_t     *large;     // 大块内存链表
    ngx_pool_cleanup_t   *cleanup;   // 清理函数链表
    ngx_log_t            *log;       // 日志对象
};
```  
其中，ngx_pool_data_t 定义了每个内存块的元数据：  
```
typedef struct {
    u_char      *last;    // 当前内存块中已使用的最后位置
    u_char      *end;     // 当前内存块的末尾位置
    ngx_pool_t  *next;    // 下一个内存块的指针
    ngx_uint_t   failed;  // 分配失败次数
} ngx_pool_data_t;
```  

## 2.2 内存池的创建和销毁
`ngx_create_pool`函数用于创建一个内存池，分配一个初始大小为 size 的内存块，并初始化内存池的各种参数：  
```
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log) {
    ngx_pool_t *p;

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);  // 对齐分配内存
    if (p == NULL) {
        return NULL;
    }

    p->d.last = (u_char *) p + sizeof(ngx_pool_t);  // 初始化last指针
    p->d.end = (u_char *) p + size;  // 指向内存块的末尾
    p->d.next = NULL;  
    p->d.failed = 0;

    size = size - sizeof(ngx_pool_t);
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;  // 确定分配块的最大值

    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```  

`ngx_destroy_pool`用于销毁内存池，会遍历内存池中的所有内存块、清理大块内存、并执行所有注册的清理函数：  
```
void ngx_destroy_pool(ngx_pool_t *pool) {
    ngx_pool_t *p, *n;
    ngx_pool_large_t *l;
    ngx_pool_cleanup_t *c;

    // 调用所有注册的清理函数
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            c->handler(c->data);
        }
    }

    // 释放大块内存
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }

    // 释放内存池中的所有内存块
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);
        if (n == NULL) {
            break;
        }
    }
}
```  

# 3. 内存池的分配策略
Nginx 内存池根据分配内存的大小来决定从哪里分配：  
小块内存（小于 `max`）：从内存池的现有块中分配。  
大块内存（大于 `max`）：直接从系统中分配内存并挂载到 `large` 链表中。  

## 3.1 小块内存分配
通过` ngx_palloc_small `函数完成。当要分配的内存块小于` max `时，它会从当前内存块中分配。  
```
static ngx_inline void *ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align) {
    u_char *m;
    ngx_pool_t *p = pool->current;

    do {
        m = p->d.last;

        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);  // 地址对齐
        }

        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;
            return m;
        }

        p = p->d.next;  // 尝试下一个内存块
    } while (p);

    return ngx_palloc_block(pool, size);  // 若当前内存块不足，则分配新内存块
}

```  
`d.last`：检查当前内存块是否有足够空间可分配。  

## 3.2 大块内存分配
当需要分配的内存块大于 `max `时，调用 `ngx_palloc_large` 函数，它直接调用系统的内存分配函数，并将大块内存挂载到` large` 链表中管理。  
```
static void *ngx_palloc_large(ngx_pool_t *pool, size_t size) {
    void *p = ngx_alloc(size, pool->log);  // 系统内存分配
    if (p == NULL) {
        return NULL;
    }

    ngx_pool_large_t *large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;  // 将大块内存挂载到 large 链表中
    large->next = pool->large;
    pool->large = large;

    return p;
}

```  

# 4. 内存池的清理机制
Nginx 内存池支持注册清理函数，在销毁内存池时自动调用这些函数进行资源清理。  

`ngx_pool_cleanup_add`函数用于注册清理函数。它会创建一个 `ngx_pool_cleanup_t `结构，并将其加入到内存池的清理链表中：  
```
ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size) {
    ngx_pool_cleanup_t *c = ngx_palloc(p, sizeof(ngx_pool_cleanup_t));
    if (c == NULL) {
        return NULL;
    }

    if (size) {
        c->data = ngx_palloc(p, size);
        if (c->data == NULL) {
            return NULL;
        }
    } else {
        c->data = NULL;
    }

    c->handler = NULL;  // 清理函数指针
    c->next = p->cleanup;
    p->cleanup = c;

    return c;
}

```