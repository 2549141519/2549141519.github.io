点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. Buffer Pool（缓冲池）介绍
Buffer Pool 是数据库管理系统中用于管理和优化内存使用的一个关键组件。其主要作用是将磁盘上的数据页缓存在内存中，以减少对磁盘的访问次数，从而提高数据库的性能。大多数现代数据库（如 MySQL、PostgreSQL、OceanBase 等）都使用了 Buffer Pool 来加速读写操作。  

## 1.1 Buffer Pool 的核心组件
Page (页)：
数据库以页（page）为单位管理数据。页是数据库中数据的最小管理单位，通常大小为4KB、8KB或16KB（具体取决于数据库的配置）。  
Buffer Pool 中的每一个缓存块都会缓存一个磁盘上的数据页，当客户端请求某个数据时，数据库首先会检查该数据页是否已经在 Buffer Pool 中，如果在，就可以直接从内存读取数据；如果不在，则需要从磁盘加载该页到 Buffer Pool 中。  
Buffer Frame (缓存帧)：  
Buffer Pool 由多个 Buffer Frame 组成，每个 Frame 都可以存储一个数据库页。当数据库需要访问某个数据页时，如果该页已经在某个 Frame 中，那么这个 Frame 就会被直接返回。否则，将分配一个空的或可重用的 Frame 用来存储从磁盘读取的页。  
Page Table (页表)：  
Buffer Pool 维护一个页表（Page Table），用于记录每个数据页在 Buffer Pool 中的位置。通过页表，数据库可以快速找到某个数据页是否已经被缓存，以及它位于 Buffer Pool 中的哪个缓存帧。  
Replacement Policy (替换策略)：  
由于内存有限，Buffer Pool 无法无限制地缓存所有数据。当 Buffer Pool 已满时，数据库需要根据某种替换策略将某些页从 Buffer Pool 中移除，以便为新加载的页腾出空间。常见的替换策略包括：  
**LRU (Least Recently Used)：最久未使用的页会被优先替换。**  
**MRU (Most Recently Used)：最近使用的页会被优先替换。**  
**LFU (Least Frequently Used)：最少被访问的页会被优先替换。**  
Dirty Page (脏页)：  
当某个页的数据被修改时，该页称为脏页。脏页表示 Buffer Pool 中的数据已经与磁盘上的数据不一致，需要在合适的时候将脏页写回磁盘以保证数据的持久性。  
数据库系统会通过后台线程定期地将脏页刷回到磁盘，或者在某些条件下强制刷新。  

## 1.2 Buffer Pool 的工作原理
读取数据：  
当数据库接收到一个查询请求时，它首先检查所需的页是否在 Buffer Pool 中。  
如果该页已经在 Buffer Pool 中，称为 "命中"（hit），系统可以直接从内存中读取数据。  
如果该页不在 Buffer Pool 中，称为 "未命中"（miss），数据库系统需要从磁盘加载该页到 Buffer Pool 中，并将其存放在某个可用的缓存帧中。  

写入数据：  
写入操作会首先修改 Buffer Pool 中相应页的内容，同时将该页标记为 "脏页"。  
数据库不会立刻将脏页写回磁盘，而是会将写入请求缓存，等到合适的时机再批量地将脏页刷回磁盘，以提高写入效率。  

页替换：  
当 Buffer Pool 已经满时，如果有新的页需要加载到 Buffer Pool 中，数据库会根据替换策略选择一个现有的页进行替换。  
如果被替换的页是脏页，则需要在替换之前将其写回到磁盘。  

# 2. Buffer Pool在miniob中的实现
disk_buffer_pool.h 和 disk_buffer_pool.cpp:  
文件定义了 [`DiskBufferPool`](#21-核心实现) 类，它是 MiniOB 中缓冲池的核心实现。`DiskBufferPool` 负责将数据页从磁盘加载到内存，并管理这些页的内存缓存。
该类提供了页面的分配 ([`allocate_page`](#212-diskbufferpoolallocate_pageframe-frame))、读取 ([`get_this_page`](#211-diskbufferpoolget_this_pagepagenum-page_num-frame-frame))、释放 ([`dispose_page`](#214-rc-diskbufferpooldispose_pagepagenum-page_num)) 以及将内存中的页面数据刷新回磁盘 ([`flush_page`](#213-diskbufferpoolflush_pageframe-frame)) 等功能。  

frame.h 和 frame.cpp:  
文件定义了 [`Frame`](#22-缓存页框架) 类，表示缓冲池中的每一个缓存页框架。`Frame` 包含页框的元数据信息（如 `buffer_pool_id`, `page_num` 等），并通过 `pin` 和 `unpin `方法管理页面的引用计数。  
`Frame `类还维护每个页面的脏页标记（`dirty_`），以及 `pin_count_` 用于防止正在使用的页面被替换。该类还负责页面的加锁与解锁操作。  

double_write_buffer.h 和 double_write_buffer.cpp:  
文件实现了双写缓冲区机制 (DoubleWriteBuffer)，用于在写入磁盘之前将页面数据写入到一个临时的缓冲区。这是一种防止部分写入（partial write）问题的机制，确保在页面写入失败时仍然能够恢复数据。  

[`BPFrameManager `](#215-bpframemanager)类:  
该类管理所有的缓冲页框架（`Frame`）。当系统需要分配新的页面或释放已有页面时，它负责在缓存池中查找或分配空闲的页面框架，并确保内存使用的有效性。   
它还实现了淘汰策略（[`purge_frames`](#23-淘汰策略) 方法），以便在内存不足时释放不常用的页面。  

## 2.1 核心实现
`DiskBufferPool`:  
```
class DiskBufferPool final
{
public:
  DiskBufferPool(BufferPoolManager &bp_manager, BPFrameManager &frame_manager, DoubleWriteBuffer &dblwr_manager,
      LogHandler &log_handler);
  ~DiskBufferPool();

  /**
   * 根据文件名打开一个分页文件
   */
  RC open_file(const char *file_name);

  /**
   * 关闭分页文件
   */
  RC close_file();

  /**
   * 根据文件ID和页号获取指定页面到缓冲区，返回页面句柄指针。
   */
  RC get_this_page(PageNum page_num, Frame **frame);

  /**
   * @brief 在指定文件中分配一个新的页面，并将其放入缓冲区，返回页面句柄指针。
   * @details 分配页面时，如果文件中有空闲页，就直接分配一个空闲页；
   * 如果文件中没有空闲页，则扩展文件规模来增加新的空闲页。
   */
  RC allocate_page(Frame **frame);

  /**
   * @brief 释放某个页面，将此页面设置为未分配状态
   *
   * @param page_num 待释放的页面
   */
  RC dispose_page(PageNum page_num);

  /**
   * @brief 释放指定文件关联的页的内存
   * 如果已经脏， 则刷到磁盘，除了pinned page
   */
  RC purge_page(PageNum page_num);
  RC purge_all_pages();

  /**
   * @brief 用于解除pageHandle对应页面的驻留缓冲区限制
   *
   * 在调用GetThisPage或AllocatePage函数将一个页面读入缓冲区后，
   * 该页面被设置为驻留缓冲区状态，以防止其在处理过程中被置换出去，
   * 因此在该页面使用完之后应调用此函数解除该限制，使得该页面此后可以正常地被淘汰出缓冲区
   */
  RC unpin_page(Frame *frame);

  /**
   * 检查是否所有页面都是pin count == 0状态(除了第1个页面)
   * 调试使用
   */
  RC check_all_pages_unpinned();

  int file_desc() const;

  /**
   * 如果页面是脏的，就将数据刷新到double write buffer
   */
  RC flush_page(Frame &frame);

  /**
   * 刷新所有页面到double write buffer，即使pin count不是0
   */
  RC flush_all_pages();

  /**
   * 回放日志时处理page0中已被认定为不存在的page
   */
  RC recover_page(PageNum page_num);

  /**
   * 刷新页面到磁盘
   */
  RC write_page(PageNum page_num, Page &page);

  RC redo_allocate_page(LSN lsn, PageNum page_num);
  RC redo_deallocate_page(LSN lsn, PageNum page_num);

public:
  int32_t id() const { return buffer_pool_id_; }

  const char *filename() const { return file_name_.c_str(); }

protected:
  RC allocate_frame(PageNum page_num, Frame **buf);

  /**
   * 刷新指定页面到磁盘(flush)，并且释放关联的Frame
   */
  RC purge_frame(PageNum page_num, Frame *used_frame);
  RC check_page_num(PageNum page_num);

  /**
   * 加载指定页面的数据到内存中
   */
  RC load_page(PageNum page_num, Frame *frame);

  /**
   * 如果页面是脏的，就将数据刷新到磁盘
   */
  RC flush_page_internal(Frame &frame);

private:
  BufferPoolManager   &bp_manager_;     /// BufferPool 管理器
  BPFrameManager      &frame_manager_;  /// Frame 管理器
  DoubleWriteBuffer   &dblwr_manager_;  /// Double Write Buffer 管理器
  BufferPoolLogHandler log_handler_;    /// BufferPool 日志处理器

  int file_desc_ = -1;  /// 文件描述符
  /// 由于在最开始打开文件时，没有正确的buffer pool id不能加载header frame，所以单独从文件中读取此标识
  int32_t       buffer_pool_id_ = -1;
  Frame        *hdr_frame_      = nullptr;  /// 文件头页面
  BPFileHeader *file_header_    = nullptr;  /// 文件头
  set<PageNum>  disposed_pages_;            /// 已经释放的页面

  string file_name_;  /// 文件名

  common::Mutex lock_;
  common::Mutex wr_lock_;

private:
  friend class BufferPoolIterator;
};
```  


### 2.1.1 `DiskBufferPool::get_this_page(PageNum page_num, Frame **frame)`
从磁盘或内存中获取指定的页面。  
```
RC DiskBufferPool::get_this_page(PageNum page_num, Frame **frame) {
  Frame *result_frame = nullptr;

  // 检查页面是否已加载在缓冲区
  result_frame = frame_manager_.get(buffer_pool_id_, page_num);
  if (result_frame != nullptr) {
    *frame = result_frame;
    result_frame->pin();
    return RC::SUCCESS;
  }

  // 如果页面不在缓冲区内，则从磁盘加载
  RC rc = allocate_frame(page_num, &result_frame);
  if (rc != RC::SUCCESS) {
    return rc;
  }

  rc = load_page(page_num, result_frame);
  if (rc != RC::SUCCESS) {
    free(buffer_pool_id_, page_num, result_frame);
    return rc;
  }

  result_frame->pin();
  *frame = result_frame;
  return RC::SUCCESS;
}
```  

首先检查页面是否已经在内存中的缓冲区（通过 `frame_manager_.get()` 方法）。如果存在，直接返回该页面，并增加 `pin` 计数。
如果页面不在内存中，调用 `allocate_frame()` 函数分配一个新的 `Frame`，并通过 `load_page()` 函数从磁盘加载该页面到内存。
成功加载页面后，增加` pin` 计数并返回。  

### 2.1.2 `DiskBufferPool::allocate_page(Frame **frame)`
分配一个新的页面到缓冲区。如果没有空闲页面，则扩展文件。  
```
RC DiskBufferPool::allocate_page(Frame **frame) {
  // 确认是否有空闲页
  if (file_header_->allocated_pages >= file_header_->page_count) {
    // 没有空闲页，扩展文件
    RC rc = expand_file();
    if (rc != RC::SUCCESS) {
      return rc;
    }
  }

  // 分配页面，更新位图状态
  PageNum page_num = file_header_->allocated_pages++;
  file_header_->bitmap[page_num / 8] |= (1 << (page_num % 8));

  // 分配帧
  return allocate_frame(page_num, frame);
}
```  

首先检查文件中是否还有空闲页，如果没有，则调用 `expand_file()` 来扩展文件。  
成功扩展文件后，通过更新 `bitmap` 位图标记页面已分配。  
使用 `allocate_frame()` 方法为该页面分配内存。   

### 2.1.3 `DiskBufferPool::flush_page(Frame &frame)`
将内存中的页面数据写回磁盘，确保页面的持久化。  
```
RC DiskBufferPool::flush_page(Frame &frame) {
  if (!frame.dirty()) {
    return RC::SUCCESS;
  }

  // 将页面数据写入磁盘
  RC rc = write_page(frame.page_num(), frame.page());
  if (rc != RC::SUCCESS) {
    return rc;
  }

  frame.clear_dirty();
  return RC::SUCCESS;
}
```  
 
先检查页面是否为脏页（`dirty()`），如果不是脏页，直接返回。  
调用 `write_page()` 方法将页面数据写入磁盘。  
写入成功后清除页面的脏标记，并返回成功状态。  

### 2.1.4 `RC DiskBufferPool::dispose_page(PageNum page_num)`
该方法用于释放某个已分配的页面，将其从缓冲区和磁盘文件中标记为未分配状态。  
```
RC DiskBufferPool::dispose_page(PageNum page_num)
{
  if (page_num == 0) {
    LOG_ERROR("Failed to dispose page %d, because it is the first page. filename=%s", page_num, file_name_.c_str());
    return RC::INTERNAL;
  }
  
  scoped_lock lock_guard(lock_);
  Frame           *used_frame = frame_manager_.get(id(), page_num);
  if (used_frame != nullptr) {
    ASSERT("the page try to dispose is in use. frame:%s", used_frame->to_string().c_str());
    frame_manager_.free(id(), page_num, used_frame);
  } else {
    LOG_DEBUG("page not found in memory while disposing it. pageNum=%d", page_num);
  }

  LSN lsn = 0;
  RC rc = log_handler_.deallocate_page(page_num, lsn);
  if (OB_FAIL(rc)) {
    LOG_ERROR("Failed to log deallocate page %d, rc=%s", page_num, strrc(rc));
    // ignore error handle
  }

  hdr_frame_->set_lsn(lsn);
  hdr_frame_->mark_dirty();
  file_header_->allocated_pages--;
  char tmp = 1 << (page_num % 8);
  file_header_->bitmap[page_num / 8] &= ~tmp;
  return RC::SUCCESS;
}
```  

检查页面号，确保文件头页面(`page_num`=0)不会被释放。   
检查页面是否在内存中，如果在内存中则释放它。  
记录日志操作，确保释放页面的操作可追溯。  
更新文件头的页面分配计数和位图，确保页面释放状态同步到文件:  
`hdr_frame_->set_lsn(lsn)`：将当前页面头文件的日志序列号（`LSN`）更新为最新的 `lsn`，确保日志顺序正确。  
`hdr_frame_->mark_dirty()`：标记页面为脏页，表示页面头已经修改，在适当时机会将其刷回磁盘。  
`file_header_->allocated_pages--`：减少文件头中记录的已分配页面的计数，表示有一个页面被释放。  
位图更新：`bitmap` 用于跟踪页面的分配状态。通过 `file_header_->bitmap[page_num / 8] &= ~tmp` 这行代码，清除对应页面的分配标记（使用按位与和按位取反操作），将页面标记为未分配。  

### 2.1.5 `BPFrameManager`
```
class BPFrameManager
{
public:
  BPFrameManager(const char *tag);

  RC init(int pool_num);
  RC cleanup();

  /**
   * @brief 获取指定的页面
   *
   * @param buffer_pool_id buffer Pool标识
   * @param page_num  页面号
   * @return Frame* 页帧指针
   */
  Frame *get(int buffer_pool_id, PageNum page_num);

  /**
   * @brief 列出所有指定文件的页面
   *
   * @param buffer_pool_id buffer Pool标识
   * @return list<Frame *> 页帧列表
   */
  list<Frame *> find_list(int buffer_pool_id);

  /**
   * @brief 分配一个新的页面
   *
   * @param buffer_pool_id buffer Pool标识
   * @param page_num 页面编号
   * @return Frame* 页帧指针
   */
  Frame *alloc(int buffer_pool_id, PageNum page_num);

  /**
   * 尽管frame中已经包含了buffer_pool_id和page_num，但是依然要求
   * 传入，因为frame可能忘记初始化或者没有初始化
   */
  RC free(int buffer_pool_id, PageNum page_num, Frame *frame);

  /**
   * 如果不能从空闲链表中分配新的页面，就使用这个接口，
   * 尝试从pin count=0的页面中淘汰一些
   * @param count 想要purge多少个页面
   * @param purger 需要在释放frame之前，对页面做些什么操作。当前是刷新脏数据到磁盘
   * @return 返回本次清理了多少个页面
   */
  int purge_frames(int count, function<RC(Frame *frame)> purger);

  size_t frame_num() const { return frames_.count(); }

  /**
   * 测试使用。返回已经从内存申请的个数
   */
  size_t total_frame_num() const { return allocator_.get_size(); }

private:
  Frame *get_internal(const FrameId &frame_id);
  RC     free_internal(const FrameId &frame_id, Frame *frame);

private:
  class BPFrameIdHasher
  {
  public:
    size_t operator()(const FrameId &frame_id) const { return frame_id.hash(); }
  };

  using FrameLruCache  = common::LruCache<FrameId, Frame *, BPFrameIdHasher>;
  using FrameAllocator = common::MemPoolSimple<Frame>;

  mutex          lock_;
  FrameLruCache  frames_;
  FrameAllocator allocator_;
};
```  


## 2.2 缓存页框架
`Frame`:  
```
class Frame
{
public:
  ~Frame()
  {
    // LOG_DEBUG("deallocate frame. this=%p, lbt=%s", this, common::lbt());
  }

  /**
   * @brief reinit 和 reset 在 MemPoolSimple 中使用
   * @details 在 MemPoolSimple 分配和释放一个Frame对象时，不会调用构造函数和析构函数，
   * 而是调用reinit和reset。
   */
  void reinit() {}
  void reset() {}

  void clear_page() { memset(&page_, 0, sizeof(page_)); }

  int  buffer_pool_id() const { return frame_id_.buffer_pool_id(); }
  void set_buffer_pool_id(int id) { frame_id_.set_buffer_pool_id(id); }

  /**
   * @brief 在磁盘和内存中内容完全一致的数据页
   * @details 磁盘文件划分为一个个页面，每次从磁盘加载到内存中，也是一个页面，就是 Page。
   * frame 是为了管理这些页面而维护的一个数据结构。
   */
  Page &page() { return page_; }

  /**
   * @brief 每个页面都有一个编号
   * @details 当前页面编号记录在了页面数据中，其实可以不记录，从磁盘中加载时记录在Frame信息中即可。
   */
  PageNum page_num() const { return frame_id_.page_num(); }
  void    set_page_num(PageNum page_num) { frame_id_.set_page_num(page_num); }
  FrameId frame_id() const { return frame_id_; }

  /**
   * @brief 为了实现持久化，需要将页面的修改记录记录到日志中，这里记录了日志序列号
   * @details 如果当前页面从磁盘中加载出来时，它的日志序列号比当前WAL(Write-Ahead-Logging)中的一些
   * 序列号要小，那就可以从日志中读取这些更大序列号的日志，做重做操作，将页面恢复到最新状态，也就是redo。
   */
  LSN  lsn() const { return page_.lsn; }
  void set_lsn(LSN lsn) { page_.lsn = lsn; }

  /**
   * @brief 页面校验和
   * @details 用于校验页面完整性。如果页面写入一半时出现异常，可以通过校验和检测出来。
   */
  CheckSum check_sum() const { return page_.check_sum; }
  void     set_check_sum(CheckSum check_sum) { page_.check_sum = check_sum; }

  /**
   * @brief 刷新当前内存页面的访问时间
   * @details 由于内存是有限的，比磁盘要小很多。那当我们访问某些文件页面时，可能由于内存不足
   * 而要淘汰一些页面。我们选择淘汰哪些页面呢？这里使用了LRU算法，即最近最少使用的页面被淘汰。
   * 最近最少使用，采用的依据就是访问时间。所以每次访问某个页面时，我们都要刷新一下访问时间。
   */
  void access();

  /**
   * @brief 标记指定页面为“脏”页。
   * @details 如果修改了页面的内容，则应调用此函数，
   * 以便该页面被淘汰出缓冲区时系统将新的页面数据写入磁盘文件
   */
  void mark_dirty() { dirty_ = true; }

  /**
   * @brief 重置“脏”标记
   * @details 如果页面已经被写入磁盘文件，则应调用此函数。
   */
  void clear_dirty() { dirty_ = false; }
  bool dirty() const { return dirty_; }

  char *data() { return page_.data; }

  bool can_purge() { return pin_count_.load() == 0; }

  /**
   * @brief 给当前页帧增加引用计数
   * pin通常都会加着frame manager锁来访问。
   * 当我们访问某个页面时，我们不期望此页面被淘汰，所以我们会增加引用计数。
   */
  void pin();

  /**
   * @brief 释放一个当前页帧的引用计数
   * 与pin对应，但是通常不会加着frame manager的锁来访问
   */
  int unpin();
  int pin_count() const { return pin_count_.load(); }

  void write_latch();
  void write_latch(intptr_t xid);

  void write_unlatch();
  void write_unlatch(intptr_t xid);

  void read_latch();
  void read_latch(intptr_t xid);
  bool try_read_latch();

  void read_unlatch();
  void read_unlatch(intptr_t xid);

  string to_string() const;

private:
  friend class BufferPool;

  bool          dirty_ = false;
  atomic<int>   pin_count_{0};
  unsigned long acc_time_ = 0;
  FrameId       frame_id_;
  Page          page_;

  /// 在非并发编译时，加锁解锁动作将什么都不做
  common::RecursiveSharedMutex lock_;

  /// 使用一些手段来做测试，提前检测出头疼的死锁问题
  /// 如果编译时没有增加调试选项，这些代码什么都不做
  common::DebugMutex           debug_lock_;
  intptr_t                     write_locker_          = 0;
  int                          write_recursive_count_ = 0;
  unordered_map<intptr_t, int> read_lockers_;
};
```  


`Frame::pin()`：  
增加页面的 `pin_count_` 计数，防止页面被淘汰。  

`Frame::unpin()`:  
减少页面的 `pin_count_` 计数，当 `pin_count_` 归零时，允许页面被淘汰。  

`Frame::mark_dirty()`:  
标记页面为脏页，表示页面已被修改，需要在合适的时机将数据写回磁盘。  

`Frame::write_latch()`:  
为当前页面加写锁，确保在写入时不被其他线程访问。  

`Frame::access()`:  
更新页面的访问时间，用于替换策略，标识页面最近一次被访问的时间。  

## 2.3 淘汰策略
```
int BPFrameManager::purge_frames(int count, function<RC(Frame *frame)> purger)
{
  lock_guard<mutex> lock_guard(lock_);

  vector<Frame *> frames_can_purge;
  if (count <= 0) {
    count = 1;
  }
  frames_can_purge.reserve(count);

  auto purge_finder = [&frames_can_purge, count](const FrameId &frame_id, Frame *const frame) {
    if (frame->can_purge()) {
      frame->pin();
      frames_can_purge.push_back(frame);
      if (frames_can_purge.size() >= static_cast<size_t>(count)) {
        return false;  // false to break the progress
      }
    }
    return true;  // true continue to look up
  };

  frames_.foreach_reverse(purge_finder);
  LOG_INFO("purge frames find %ld pages total", frames_can_purge.size());

  /// 当前还在frameManager的锁内，而 purger 是一个非常耗时的操作
  /// 他需要把脏页数据刷新到磁盘上去，所以这里会极大地降低并发度
  int freed_count = 0;
  for (Frame *frame : frames_can_purge) {
    RC rc = purger(frame);
    if (RC::SUCCESS == rc) {
      free_internal(frame->frame_id(), frame);
      freed_count++;
    } else {
      frame->unpin();
      LOG_WARN("failed to purge frame. frame_id=%s, rc=%s", 
               frame->frame_id().to_string().c_str(), strrc(rc));
    }
  }
  LOG_INFO("purge frame done. number=%d", freed_count);
  return freed_count;
}
```  

查找可淘汰的页面:  
定义一个 `purge_finder` lambda 函数，该函数用于遍历缓存中的所有页面，并将符合条件（可以淘汰）的页面存储到 `frames_can_purge` 容器中。  
`frame->can_purge()`：判断页面是否可以被淘汰，通常是通过 `pin_count == 0` 来判断页面是否正在使用，如果页面未被引用，则可以淘汰。  
`frame->pin()`：对该页面执行 `pin` 操作，确保在当前线程使用该页面时，它不会被其他线程替换或释放。  
如果已经找到了足够数量的页面（即` frames_can_purge.size() >= count`），则返回 `false`，以终止遍历。  
`frames_.foreach_reverse(purge_finder)`：遍历缓存池中的页面，查找可以被淘汰的页面，并存储在 `frames_can_purge` 中。该遍历是逆序进行的（`foreach_reverse`），可能是因为最近使用的页面排在队列的前面，而最近未使用的页面排在后面。  

执行淘汰操作:   
初始化 `freed_count` 变量，用于统计成功淘汰的页面数量。  
遍历 `frames_can_purge` 中的页面，并为每个页面调用传入的 `purger(frame)` 函数执行淘汰前的必要操作。  
`purger(frame)`：这是一个用户提供的函数，通常用于将脏页（修改过但尚未刷盘的页面）写回磁盘。这个操作可能非常耗时，因此是关键步骤。  
如果 `purger(frame)` 成功（返回 `RC::SUCCESS`），则调用 `free_internal()` 方法释放该页面，并增加已释放页面的计数 `freed_count`。  
如果 `purger(frame)` 失败，调用 `frame->unpin()` 取消之前对页面的 `pin` 操作，并记录警告日志。  