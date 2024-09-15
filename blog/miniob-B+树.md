点击返回[🔗我的博客文章目录](https://2549141519.github.io/#/toc)
* 目录
{:toc}
<div onclick="window.scrollTo({top:0,behavior:'smooth'});" style="background-color:white;position:fixed;bottom:20px;right:40px;padding:10px 10px 5px 10px;cursor:pointer;z-index:10;border-radius:13%;box-shadow:0.5px 3px 7px rgba(0,0,0,0.3);"><img src="https://2549141519.github.io/blogImg/backTop.png" alt="TOP" style="background-color:white;width:30px;"></div>

# 1. B+树介绍
B+树是一种平衡的树形数据结构，通常用于数据库和文件系统等需要频繁进行磁盘存取的大型数据管理系统。它是B树的扩展版本，具有更高效的查询和范围查询能力。B+树的设计初衷是为了优化磁盘存取的效率，因而在处理大规模数据时非常适合。  

## 1.1 B+树的结构特点
所有关键字都出现在叶子节点：  
B+树与B树的主要区别之一是，B+树中所有的关键字都存储在叶子节点中，内部节点只存储索引信息。叶子节点按照关键字的顺序排列，并通过指针连接，形成一个有序的链表。  

内节点只存储索引而不存储数据：   
B+树的非叶子节点（内节点）不存储实际数据，它们只是用于指引查询路径的索引，指向子树或叶子节点。  

所有叶子节点在同一层：  
B+树是高度平衡的，即所有叶子节点都在同一层，确保查询操作的时间复杂度为`O(log n)`，其中`n`是数据项的总数。  

叶子节点链表结构：  
叶子节点之间是有序链表，意味着在进行范围查询时，可以通过叶子节点的顺序访问快速获取一组连续的结果。  

## 1.2 B+树的优点
范围查询性能优异：  
由于叶子节点之间的链表结构，B+树在处理范围查询时表现极为优秀，可以从起始位置沿着链表顺序读取。  

磁盘I/O优化：  
B+树的设计目标是减少磁盘I/O操作。内节点不存储实际数据，因此可以将更多的索引节点放入内存中，减少从磁盘读取数据的次数。  

平衡性：  
B+树是一种平衡树，保证了插入、删除、查询操作的时间复杂度始终保持在`O(log n)`。   

存储密度高：  
由于内节点不存储数据，B+树可以将更多的索引节点放入内存，从而有效利用系统的缓存。  

## 1.3 B+树的应用场景
数据库系统：  
在关系型数据库中，B+树常用于实现索引，如MySQL中的InnoDB存储引擎就使用B+树作为其主键索引结构。  

文件系统：  
一些文件系统，如NTFS，也采用B+树来管理文件和目录数据。  

大规模数据存储：   
B+树特别适合需要在磁盘中管理大规模数据的场景，能够很好地处理磁盘I/O。  

## 1.4 B+树与B树的区别
数据存储位置：  
B树的数据既存储在内节点也存储在叶子节点，而B+树的数据只存储在叶子节点。  

范围查询效率：  
由于B+树的叶子节点构成链表结构，进行范围查询时可以直接从链表中遍历，而B树则需要更多的树遍历操作。  

节点结构：  
B+树的内节点更为紧凑，因为它们只存储索引，而B树的内节点存储索引和数据。  

# 2. B+树在miniob的应用

## 2.1 B+树的结构与节点

### 2.1.1 `IndexFileHeader`
`IndexFileHeader`：  
用于存储 B+ 树的元数据，包括根节点页面编号、内部节点和叶子节点的最大大小、属性长度等。  
```
struct IndexFileHeader
{
  IndexFileHeader()
  {
    memset(this, 0, sizeof(IndexFileHeader));
    root_page = BP_INVALID_PAGE_NUM;
  }
  PageNum  root_page;          ///< 根节点在磁盘中的页号
  int32_t  internal_max_size;  ///< 内部节点最大的键值对数
  int32_t  leaf_max_size;      ///< 叶子节点最大的键值对数
  int32_t  attr_length;        ///< 键值的长度
  int32_t  key_length;         ///< attr length + sizeof(RID)
  AttrType attr_type;          ///< 键值的类型

  const string to_string() const
  {
    stringstream ss;

    ss << "attr_length:" << attr_length << ","
       << "key_length:" << key_length << ","
       << "attr_type:" << attr_type_to_string(attr_type) << ","
       << "root_page:" << root_page << ","
       << "internal_max_size:" << internal_max_size << ","
       << "leaf_max_size:" << leaf_max_size << ";";

    return ss.str();
  }
};
```  
`PageNum root_page`：  
该变量存储了根节点在磁盘中的页号，用于指示B+树在文件系统中的存储位置。`BP_INVALID_PAGE_NUM` 表示一个无效的页号。  
`int32_t internal_max_size`：  
这是内部节点中可以存储的最大键值对数。对于一个B+树的内部节点而言，它存储的是索引信息和指向子节点的指针，因此这个值决定了内部节点的分裂和合并策略。  
`int32_t leaf_max_size`：  
叶子节点可以存储的最大键值对数。叶子节点中存储的是实际的数据，定义了B+树中叶子节点的存储容量，影响了B+树的深度和查找效率。  
`int32_t attr_length`：  
键值的长度，通常是指数据库中索引字段的长度。它决定了每个键在树中占据的空间大小。  
`int32_t key_length`：  
该变量表示键值的总长度，即 `attr_length` 加上 `sizeof(RID)`，其中 `RID `是数据库中的记录标识符（Record ID）。这样可以存储索引键及其对应的记录位置。  
`AttrType attr_type`：  
键值的类型，定义了索引所依赖的属性类型（如整数、字符串等）。该枚举类型用于描述键值的实际数据类型。   

`to_string `函数:  
它将结构体的关键字段都格式化为一个字符串，输出属性长度、键长度、属性类型、根页号、内部节点和叶子节点的最大容量等信息,方便调试或输出。    

### 2.1.2 `IndexNode`
`IndexNode`：  
通用的 B+ 树节点结构，包括页面类型、键值对数量以及父节点的页面编号。  
```
struct IndexNode
{
  static constexpr int HEADER_SIZE = 12;

  bool    is_leaf;  /// 当前是叶子节点还是内部节点
  int     key_num;  /// 当前页面上一共有多少个键值对
  PageNum parent;   /// 父节点页面编号
};
```  

### 2.1.3 `LeafIndexNode`
`LeafIndexNode`：  
叶子节点的结构，存储实际的数据和指向下一个兄弟节点的指针。  
```
struct LeafIndexNode : public IndexNode
{
  static constexpr int HEADER_SIZE = IndexNode::HEADER_SIZE + 4;

  PageNum next_brother;

  char array[0];
};
```    

### 2.1.4 `InternalIndexNode`
`InternalIndexNode`：  
内部节点的结构，存储键值和子页面编号。  
```
struct InternalIndexNode : public IndexNode
{
  static constexpr int HEADER_SIZE = IndexNode::HEADER_SIZE;

  char array[0];
};  
```  

## 2.2 B+树的操作

### 2.2.1 插入操作
通过 `insert_entry` 方法将键值对插入 B+ 树中。如果叶子节点或内部节点满了，调用 [`split`](#241-分裂) 方法进行节点分裂。  
```
RC BplusTreeHandler::insert_entry(const char *user_key, const RID *rid)
{
  if (user_key == nullptr || rid == nullptr) {
    LOG_WARN("Invalid arguments, key is empty or rid is empty");
    return RC::INVALID_ARGUMENT;
  }

  MemPoolItem::item_unique_ptr pkey = make_key(user_key, *rid);
  if (pkey == nullptr) {
    LOG_WARN("Failed to alloc memory for key.");
    return RC::NOMEM;
  }

  RC rc = RC::SUCCESS;

  BplusTreeMiniTransaction mtr(*this, &rc);

  char *key = static_cast<char *>(pkey.get());

  if (is_empty()) {
    root_lock_.lock();
    if (is_empty()) {
      rc = create_new_tree(mtr, key, rid);
      root_lock_.unlock();
      return rc;
    }
    root_lock_.unlock();
  }

  Frame *frame = nullptr;

  rc = find_leaf(mtr, BplusTreeOperationType::INSERT, key, frame);
  if (OB_FAIL(rc)) {
    LOG_WARN("Failed to find leaf %s. rc=%d:%s", rid->to_string().c_str(), rc, strrc(rc));
    return rc;
  }

  rc = insert_entry_into_leaf_node(mtr, frame, key, rid);
  if (OB_FAIL(rc)) {
    LOG_TRACE("Failed to insert into leaf of index, rid:%s. rc=%s", rid->to_string().c_str(), strrc(rc));
    return rc;
  }

  LOG_TRACE("insert entry success");
  return RC::SUCCESS;
}
```  
`const char *user_key`：  
用户提供的键值，表示需要插入到B+树中的数据。  
`const RID *rid`：  
记录标识符（Record ID），用于唯一标识一条记录在数据库中的位置。每个`user_key`都有一个对应的`rid`。   

输入检查：  
在插入操作之前，首先检查 `user_key` 和` rid `是否为空，确保输入参数的合法性。如果为空，则记录一个警告日志并返回 `RC::INVALID_ARGUMENT` 错误码。  
创建键的副本：  
使用 `make_key` 函数将 `user_key` 和 `rid` 组合成一个键值对，并使用智能指针 `MemPoolItem::item_unique_ptr` 来管理其内存。如果内存分配失败，返回 `RC::NOMEM`。  
`BplusTreeMiniTransaction mtr(*this, &rc);`:  
创建一个 `BplusTreeMiniTransaction` 对象，这个对象通常用于管理B+树的事务操作，确保在插入操作过程中的一致性和并发控制。`mtr` 表示事务对象，它会跟踪并处理插入过程中可能发生的错误。  
空树处理：  
如果当前B+树为空（即还没有根节点），使用 `root_lock_` 锁定根节点，并调用 `create_new_tree` 来创建一个新树结构。这种情况下，插入操作直接返回创建树的结果。`is_empty()` 判断树是否为空，通过双重检查锁机制避免重复创建树。  
查找叶子节点：  
使用` find_leaf `函数根据 `key `找到适合插入的叶子节点。`find_leaf` 是查找叶子节点的核心操作之一，它将遍历B+树直到找到包含此键的叶子节点。如果查找失败（`OB_FAIL(rc)`），记录警告日志并返回相应的错误码。  
插入叶子节点：  
找到目标叶子节点后，调用 `insert_entry_into_leaf_node` 函数，将 `key `和 `rid` 插入到叶子节点中。  

### 2.2.2 删除操作
通过 `delete_entry` 方法删除键值对。删除时如果节点数量小于最小容量，则可能需要合并相邻节点或重新分配节点中的元素。  
```
RC BplusTreeHandler::delete_entry_internal(BplusTreeMiniTransaction &mtr, Frame *leaf_frame, const char *key)
{
  LeafIndexNodeHandler leaf_index_node(mtr, file_header_, leaf_frame);

  const int remove_count = leaf_index_node.remove(key, key_comparator_);
  if (remove_count == 0) {
    LOG_TRACE("no data need to remove");
    // disk_buffer_pool_->unpin_page(leaf_frame);
    return RC::RECORD_NOT_EXIST;
  }
  // leaf_index_node.validate(key_comparator_, disk_buffer_pool_, file_id_);

  leaf_frame->mark_dirty();

  if (leaf_index_node.size() >= leaf_index_node.min_size()) {
    return RC::SUCCESS;
  }

  return coalesce_or_redistribute<LeafIndexNodeHandler>(mtr, leaf_frame);
}

RC BplusTreeHandler::delete_entry(const char *user_key, const RID *rid)
{
  MemPoolItem::item_unique_ptr pkey = mem_pool_item_->alloc_unique_ptr();
  if (nullptr == pkey) {
    LOG_WARN("Failed to alloc memory for key. size=%d", file_header_.key_length);
    return RC::NOMEM;
  }
  char *key = static_cast<char *>(pkey.get());

  memcpy(key, user_key, file_header_.attr_length);
  memcpy(key + file_header_.attr_length, rid, sizeof(*rid));

  BplusTreeOperationType op = BplusTreeOperationType::DELETE;

  RC rc = RC::SUCCESS;

  BplusTreeMiniTransaction mtr(*this, &rc);

  Frame *leaf_frame = nullptr;

  rc = find_leaf(mtr, op, key, leaf_frame);
  if (rc == RC::EMPTY) {
    rc = RC::RECORD_NOT_EXIST;
    return rc;
  }

  if (OB_FAIL(rc)) {
    LOG_WARN("failed to find leaf page. rc =%s", strrc(rc));
    return rc;
  }

  rc = delete_entry_internal(mtr, leaf_frame, key);
  return rc;
}
```  
`delete_entry`函数：B+树删除操作的入口。  
内存分配并构造删除键：  
首先从内存池 `mem_pool_item_` 分配一个指向 `key `的内存指针。如果分配失败，则记录警告日志并返回` RC::NOMEM `错误码。  
然后将` user_key `和` rid `组合成一个键，存储在` key `变量中。这一步是将用户提供的键和记录ID合并成一个可以在B+树中查找和删除的完整键。  
查找叶子节点：  
创建一个 `BplusTreeMiniTransaction` 来保证B+树在操作中的一致性。  
使用` find_leaf `函数根据` key `找到目标叶子节点所在的页框` leaf_frame`。  
如果返回值为 `RC::EMPTY`，表示B+树中没有数据，直接返回 `RC::RECORD_NOT_EXIST` 表示记录不存在。   

`delete_entry_internal`函数：处理删除操作的核心逻辑。  
删除键值对：  
使用 `LeafIndexNodeHandler` 来处理叶子节点的操作，并通过 `remove` 函数尝试删除目标键。  
`remove_count` 记录删除的键值对数量，如果为0，表示没有找到需要删除的键，记录日志并返回` RC::RECORD_NOT_EXIST`。  
标记叶子节点为`dirty`：`leaf_frame->mark_dirty();`  
删除成功后，将叶子节点标记为“脏”，即需要写回磁盘。  
检查节点是否需要合并或重分配：  
检查叶子节点的当前大小是否大于等于最小值 `min_size`，如果是，表示节点仍然保持平衡，不需要进一步处理，返回 `RC::SUCCESS`。  
如果节点的大小小于最小值，调用 `coalesce_or_redistribute` 函数，尝试将节点与相邻节点合并或从相邻节点中重分配一些数据，以维持B+树的平衡。  

### 2.2.3 查找操作
`get_entry `方法用于查找指定的键值对，`find_leaf` 用于找到包含指定键值的叶子节点。  
```  
RC BplusTreeHandler::get_entry(const char *user_key, int key_len, list<RID> &rids)
{
  BplusTreeScanner scanner(*this);
  RC rc = scanner.open(user_key, key_len, true /*left_inclusive*/, user_key, key_len, true /*right_inclusive*/);
  if (OB_FAIL(rc)) {
    LOG_WARN("failed to open scanner. rc=%s", strrc(rc));
    return rc;
  }

  RID rid;
  while ((rc = scanner.next_entry(rid)) == RC::SUCCESS) {
    rids.push_back(rid);
  }

  scanner.close();
  if (rc != RC::RECORD_EOF) {
    LOG_WARN("scanner return error. rc=%s", strrc(rc));
  } else {
    rc = RC::SUCCESS;
  }
  return rc;
}

RC BplusTreeHandler::find_leaf(BplusTreeMiniTransaction &mtr, BplusTreeOperationType op, const char *key, Frame *&frame)
{
  auto child_page_getter = [this, key](InternalIndexNodeHandler &internal_node) {
    return internal_node.value_at(internal_node.lookup(key_comparator_, key));
  };
  return find_leaf_internal(mtr, op, child_page_getter, frame);
}
```  

`get_entry `函数:  
用于根据给定的` user_key `从B+树中找到对应的记录ID列表` rids`，它主要使用了一个[` BplusTreeScanner `](#25-b树的扫描)来扫描树中的符合条件的键。  
初始化` BplusTreeScanner`：  
首先初始化一个 `BplusTreeScanner` 对象，这个对象负责遍历和扫描B+树的节点。  
调用 `scanner.open()` 来设置扫描的范围。这里扫描范围设定为从` user_key `开始，`left_inclusive` 和 `right_inclusive` 表示是否包含边界键，即此处是查找等于 `user_key` 的条目。  
遍历并获取记录ID：  
使用 `scanner.next_entry(rid)` 不断获取下一个满足条件的 `RID`。每次成功获取到 `RID` 后，将其添加到 `rids` 列表中。  
关闭扫描器并返回结果：  
在完成扫描后，调用` scanner.close() `关闭扫描器。  

`find_leaf `函数:  
用于在B+树中查找包含特定` key `的叶子节点，它通过递归遍历树的内部节点，直到找到目标叶子节点。  
查找子节点页号的回调函数:    
定义一个` child_page_getter `lambda 函数，用于从内部节点获取子节点的页号。  
`internal_node.lookup(key_comparator_, key)` 会根据 `key `使用 `key_comparator_ `来查找在内部节点中的位置，并通过` value_at() `获取对应子节点的页号。  
调用 `find_leaf_internal`：  
传入事务对象 `mtr`、操作类型 `op`（如删除、插入等）、获取子节点页号的 `child_page_getter` 以及页框 `frame`。  

## 2.3 B+树的事务
`BplusTreeMiniTransaction`：用于封装 MiniOB 中的 B+ 树事务。  
```  
class BplusTreeMiniTransaction final
{
public:

  BplusTreeMiniTransaction(BplusTreeHandler &tree_handler, RC *operation_result = nullptr);
  ~BplusTreeMiniTransaction();

  LatchMemo       &latch_memo() { return latch_memo_; }
  BplusTreeLogger &logger() { return logger_; }

  RC commit();
  RC rollback();

private:
  BplusTreeHandler &tree_handler_;
  RC               *operation_result_ = nullptr;
  LatchMemo         latch_memo_;
  BplusTreeLogger   logger_;
};
```  
`BplusTreeMiniTransaction(BplusTreeHandler &tree_handler, RC *operation_result = nullptr)`：  
构造函数，接收一个` BplusTreeHandler `对象作为参数，这个对象代表了对 B+ 树的具体操作管理。  
参数 operation_result 是一个指向 `RC`（返回码）的指针。如果不为 `nullptr`，在事务结束时，会根据操作结果来决定是否提交或回滚。  
该构造函数通常会在事务开始时被调用，负责初始化事务对象，准备后续的操作。  
`~BplusTreeMiniTransaction()`：  
析构函数，负责清理事务资源。如果在事务结束之前没有显式地调用 `commit()` 或 `rollback()`，析构函数可能会处理未决的事务，确保不会有未提交或未回滚的操作残留。  
`RC commit()`：  
该函数用于提交事务。当事务中的所有操作都成功执行时，调用 `commit()` 会将操作的结果提交到B+树，并将相关的日志清除或标记为已完成。  
`RC rollback()`：  
该函数用于回滚事务。在事务执行过程中，如果发生错误或操作失败，调用 `rollback()` 会撤销事务中的所有修改，恢复到操作前的状态。  
回滚操作使用 `BplusTreeLogger` 中记录的日志来还原之前的状态。  

`tree_handler_` 是对 B+ 树操作的管理器，它负责对 B+ 树执行实际的操作。事务类通过与 `tree_handler_` 交互来进行树的插入、删除、查找等操作。  
`operation_result_` 是一个返回码的指针，用来标记事务执行过程中的状态。如果这个指针不为空，事务结束时会根据它的值来自动决定是否提交或回滚。  
`latch_memo_` 是一个锁定机制的记忆记录对象,跟踪事务期间获取的锁定资源，确保在事务结束时能够正确释放资源，避免死锁或资源泄漏。  
`logger_ `是事务操作的日志记录器，它会记录事务期间对 B+ 树进行的所有操作，以便在事务回滚时能够还原数据，或者在崩溃恢复时重新应用事务。  

## 2.4 B+树的分裂与合并

### 2.4.1 分裂
当节点满时，`split `方法将节点分裂成两个，并将中间的键插入到父节点。  
```
template <typename IndexNodeHandlerType>
RC BplusTreeHandler::split(BplusTreeMiniTransaction &mtr, Frame *frame, Frame *&new_frame)
{
  IndexNodeHandlerType old_node(mtr, file_header_, frame);

  // add a new node
  RC rc = mtr.latch_memo().allocate_page(new_frame);
  if (OB_FAIL(rc)) {
    LOG_WARN("Failed to split index page due to failed to allocate page, rc=%d:%s", rc, strrc(rc));
    return rc;
  }

  mtr.latch_memo().xlatch(new_frame);

  IndexNodeHandlerType new_node(mtr, file_header_, new_frame);
  new_node.init_empty();
  new_node.set_parent_page_num(old_node.parent_page_num());

  old_node.move_half_to(new_node);

  frame->mark_dirty();
  new_frame->mark_dirty();
  return RC::SUCCESS;
}
```  
分配新页：为新节点分配页框并加锁。  
移动数据：将旧节点的一半数据移动到新节点。  
更新元数据：确保新节点继承旧节点的父节点关系。  
标记修改：标记修改后的页框，以便事务提交时更新磁盘中的数据。  

### 2.4.2 合并
当删除导致节点中的元素过少时，`coalesce` 方法合并相邻节点。  
```
template <typename IndexNodeHandlerType>
RC BplusTreeHandler::coalesce(
    BplusTreeMiniTransaction &mtr, Frame *neighbor_frame, Frame *frame, Frame *parent_frame, int index)
{
  InternalIndexNodeHandler parent_node(mtr, file_header_, parent_frame);

  // 先区分出来左右节点
  Frame *left_frame  = nullptr;
  Frame *right_frame = nullptr;
  if (index == 0) {
    // neighbor node is at right
    left_frame  = frame;
    right_frame = neighbor_frame;
    index++;
  } else {
    left_frame  = neighbor_frame;
    right_frame = frame;
    // neighbor is at left
  }

  // 把右边节点的数据复制到左边节点上去
  IndexNodeHandlerType left_node(mtr, file_header_, left_frame);
  IndexNodeHandlerType right_node(mtr, file_header_, right_frame);

  parent_node.remove(index);
  // parent_node.validate(key_comparator_, disk_buffer_pool_, file_id_);
  RC rc = right_node.move_to(left_node);
  if (OB_FAIL(rc)) {
    LOG_WARN("failed to move right node to left. rc=%d:%s", rc, strrc(rc));
    return rc;
  }
  // left_node.validate(key_comparator_);

  // 叶子节点维护next_page指针
  if (left_node.is_leaf()) {
    LeafIndexNodeHandler left_leaf_node(mtr, file_header_, left_frame);
    LeafIndexNodeHandler right_leaf_node(mtr, file_header_, right_frame);
    left_leaf_node.set_next_page(right_leaf_node.next_page());
  }

  // 释放右边节点
  mtr.latch_memo().dispose_page(right_frame->page_num());

  // 递归的检查父节点是否需要做合并或者重新分配节点数据
  return coalesce_or_redistribute<InternalIndexNodeHandler>(mtr, parent_frame);
}
```  
确定左右节点：根据 `index` 来区分当前节点和相邻节点（左或右）。  
删除父节点中的索引：移除父节点中与右节点对应的索引键。  
合并左右节点：将右节点中的数据移动到左节点中，如果是叶子节点，更新其` next_page` 指针。  
释放右节点：合并后释放右节点的资源。  
递归处理父节点：合并完成后递归检查父节点，看看是否需要进一步的合并或重新分配。   

### 2.4.3 重新分配
`redistribute `方法重新分配相邻节点中的元素以保持平衡。   
```
template <typename IndexNodeHandlerType>
RC BplusTreeHandler::redistribute(BplusTreeMiniTransaction &mtr, Frame *neighbor_frame, Frame *frame, Frame *parent_frame, int index)
{
  InternalIndexNodeHandler parent_node(mtr, file_header_, parent_frame);
  IndexNodeHandlerType     neighbor_node(mtr, file_header_, neighbor_frame);
  IndexNodeHandlerType     node(mtr, file_header_, frame);
  if (neighbor_node.size() < node.size()) {
    LOG_ERROR("got invalid nodes. neighbor node size %d, this node size %d", neighbor_node.size(), node.size());
  }
  if (index == 0) {
    // the neighbor is at right
    neighbor_node.move_first_to_end(node);
    // neighbor_node.validate(key_comparator_, disk_buffer_pool_, file_id_);
    // node.validate(key_comparator_, disk_buffer_pool_, file_id_);
    parent_node.set_key_at(index + 1, neighbor_node.key_at(0));
    // parent_node.validate(key_comparator_, disk_buffer_pool_, file_id_);
  } else {
    // the neighbor is at left
    neighbor_node.move_last_to_front(node);
    // neighbor_node.validate(key_comparator_, disk_buffer_pool_, file_id_);
    // node.validate(key_comparator_, disk_buffer_pool_, file_id_);
    parent_node.set_key_at(index, node.key_at(0));
    // parent_node.validate(key_comparator_, disk_buffer_pool_, file_id_);
  }

  neighbor_frame->mark_dirty();
  frame->mark_dirty();
  parent_frame->mark_dirty();

  return RC::SUCCESS;
}
```   

初始化节点和检查节点大小：准备好相邻节点、当前节点和父节点，确保相邻节点有足够的元素可以借出。  
根据相邻节点的位置重新分配元素：  
如果相邻节点在右侧，将右节点的第一个元素移动到当前节点的末尾，并更新父节点键值。  
如果相邻节点在左侧，将左节点的最后一个元素移动到当前节点的开头，并更新父节点键值。  
标记修改：标记所有被修改的节点页框，以便后续提交到磁盘。  
返回结果：操作成功后返回 `RC::SUCCESS`。  

## 2.5 B+树的扫描
`BplusTreeScanner `类允许对 B+ 树的范围扫描。可以设置左边界和右边界，扫描满足条件的键值对。  
```
class BplusTreeScanner
{
public:
  BplusTreeScanner(BplusTreeHandler &tree_handler);
  ~BplusTreeScanner();

  /**
   * @brief 扫描指定范围的数据
   * @param left_user_key 扫描范围的左边界，如果是null，则没有左边界
   * @param left_len left_user_key 的内存大小(只有在变长字段中才会关注)
   * @param left_inclusive 左边界的值是否包含在内
   * @param right_user_key 扫描范围的右边界。如果是null，则没有右边界
   * @param right_len right_user_key 的内存大小(只有在变长字段中才会关注)
   * @param right_inclusive 右边界的值是否包含在内
   * TODO 重构参数表示方法
   */
  RC open(const char *left_user_key, int left_len, bool left_inclusive, const char *right_user_key, int right_len,
      bool right_inclusive);

  /**
   * @brief 获取下一条记录
   *
   * @param rid 当前默认所有值都是RID类型。对B+树来说并不是一个好的抽象
   * @return RC RECORD_EOF 表示遍历完成
   * TODO 需要增加返回 key 的接口
   * @warning 不要在遍历时删除数据。删除数据会导致遍历器失效。
   * 当前默认的走索引删除的逻辑就是这样做的，所以删除逻辑有BUG。
   */
  RC next_entry(RID &rid);

  /**
   * @brief 关闭当前扫描器
   * @details 可以不调用，在析构函数时会自动执行
   */
  RC close();

private:
  /**
   * 如果key的类型是CHARS, 扩展或缩减user_key的大小刚好是schema中定义的大小
   */
  RC fix_user_key(const char *user_key, int key_len, bool want_greater, char **fixed_key, bool *should_inclusive);

  void fetch_item(RID &rid);

  /**
   * @brief 判断是否到了扫描的结束位置
   */
  bool touch_end();

private:
  bool                     inited_ = false;
  BplusTreeHandler        &tree_handler_;
  BplusTreeMiniTransaction mtr_;

  /// 使用左右叶子节点和位置来表示扫描的起始位置和终止位置
  /// 起始位置和终止位置都是有效的数据
  Frame *current_frame_ = nullptr;

  common::MemPoolItem::item_unique_ptr right_key_;
  int                                  iter_index_    = -1;
  bool                                 first_emitted_ = false;
};
```  
初始化扫描器：  
创建 `BplusTreeScanner `对象并调用` open `函数，设置扫描的范围（左边界和右边界）。  
获取记录：  
调用` next_entry `函数获取一条记录，并循环调用直到返回` RC::RECORD_EOF`，表示遍历完成。  
关闭扫描器：  
扫描完成后调用` close `函数关闭扫描器。关闭扫描器会释放相关资源，如果未显式调用，也会在析构函数中自动关闭。  

## 2.6 B+树的验证
通过 `validate_tree `方法可以检查 B+ 树的合法性，包括节点之间的链接、键值的顺序等。  
```
bool BplusTreeHandler::validate_tree()
{
  if (is_empty()) {
    return true;
  }

  BplusTreeMiniTransaction mtr(*this);
  LatchMemo &latch_memo = mtr.latch_memo();
  Frame    *frame = nullptr;

  RC rc = latch_memo.get_page(file_header_.root_page, frame);  // 这里仅仅调试使用，不加root锁
  if (OB_FAIL(rc)) {
    LOG_WARN("failed to fetch root page. page id=%d, rc=%d:%s", file_header_.root_page, rc, strrc(rc));
    return false;
  }

  if (!validate_node_recursive(mtr, frame) || !validate_leaf_link(mtr)) {
    LOG_WARN("Current B+ Tree is invalid");
    print_tree();
    return false;
  }

  LOG_INFO("great! current tree is valid");
  return true;
}
```  
结构验证：  
`validate_tree `通过递归检查每个节点的结构（`validate_node_recursive`）和叶子节点的链表链接（`validate_leaf_link`），确保B+树的各个部分符合规范。  
调试工具：  
当树结构无效时，函数会打印整个树的结构，便于开发者进行调试和问题排查。  