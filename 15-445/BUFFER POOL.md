### LRU替换策略

此组件负责跟踪缓冲池中的页面使用情况。在`src/include/buffer/lru_replacer.h`中实现名为`LRUReplacer`的子类，并在`src/buffer/lru_replacer.cpp`中实现其相应的实现文件。`LRUReplacer`扩展了包含函数规范的抽象类`Replacer`（`src/include/buffer/Replacer.h`）。
`LRUReplacer`的最大页数与缓冲池的大小相同，因为它包含`BufferPoolManager`中所有帧的占位符。然而，在任何给定的时刻，并非所有的帧都被认为是在`LRUReplacer`中。`LRUReplacer`初始化时没有帧。然后，只有新未固定的帧才会被视为在`LRUReplacer`中。
需要实现以下方法：
`Victim（frame_id_t*`）：删除最早加入replacer的页帧，并返回True。如果`Replacer`为空，则返回False。
`Pin（frame_id_t）`：它应该从`LRUReplacer`中删除包含固定页面的帧。
`Unpin（frame_id_t）`：当页面的`pin_count`变为0时，调用这个方法页帧添加到`LRUReplacer`。
`Size（）`：此方法返回`LRUReplacer`中当前的帧数。

1. #### 数据结构 

   LRU替换策略需要在空间满时将最久没被使用的页替换出去，选择用双向链表List存储没被使用的页，队首存储的是最久没被使用的页，队尾则是最近刚被放弃使用的页，正在使用的页不被存储在List里。当需要使用没被使用的页时，可以从List删除该页的结点。为了提高删除的速度，可以用map存储每个页对应结点的迭代器，实现O(1)时间复杂度的快速删除。

   ```c++
   private:
     // TODO(student): implement me!
     /** 缓冲区中页的数量 */
     size_t num_pages_;
     /** 存放可以被换出的帧号 */
     std::list<frame_id_t> lru_frame_list_;
     /** 保存每个可换出帧对应的list里的迭代器 */
     std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> iterator_table_;
     /** 保护共享数据frame_list和iterator_table*/
     std::mutex latch_;
   ```

   

2. #### Victim函数

   调用这个函数表示需要选择一个页被替换出去，删除队首的结点和map中相应的迭代器即可。当没有没被使用的页时，需要返回false。为了多线程并发时程序仍然正确，需要加锁。

   ```c++
   bool LRUReplacer::Victim(frame_id_t *frame_id) {
     latch_.lock();
     if (lru_frame_list_.empty()) {
       latch_.unlock();
       return false;
     }
     iterator_table_.erase(lru_frame_list_.front());
     *frame_id = lru_frame_list_.front();
     lru_frame_list_.pop_front();
     latch_.unlock();
     return true;
   }
   ```

   

3. #### Pin函数

   当没被使用的页需要重新使用时，该页就不能被替换出去，从List和Map中删除该页就行。为了多线程并发时程序仍然正确，需要加锁。

   ```c++
   void LRUReplacer::Pin(frame_id_t frame_id) {
     latch_.lock();
     if (iterator_table_.find(frame_id) == iterator_table_.end()) {
       latch_.unlock();
       return;
     }
     auto iterator = iterator_table_[frame_id];
     lru_frame_list_.erase(iterator);
     iterator_table_.erase(frame_id);
     latch_.unlock();
   }
   ```

   

4. #### Unpin函数

   当一个页`pin_count`变成0，表示该页不需要被使用了，可以被替换出去，所以将该页加到List的队尾。

   **注意**：如果一个页已经在List时，调用unpin函数时不需要把它追加到队尾，什么都不要做。
   
   为了多线程并发时程序仍然正确，需要加锁。
   
   ```c++
   void LRUReplacer::Unpin(frame_id_t frame_id) {
     latch_.lock();
     if (iterator_table_.find(frame_id) != iterator_table_.end() || lru_frame_list_.size() > num_pages_) {
       latch_.unlock();
       return;
     }
     lru_frame_list_.push_back(frame_id);
     auto iterator = lru_frame_list_.end();
     --iterator;
     iterator_table_[frame_id] = iterator;
     latch_.unlock();
   }
   ```



### 缓冲池管理器实例

接下来，您需要在系统中实现缓冲池管理器（`BufferPoolManagerInstance`）。`BufferPoolManagerInstance`负责从`DiskManager`获取数据库页并将其存储在内存中。当显式指示`BufferPoolManagerInstance`将脏页写入磁盘时，或者当它需要逐出页以为新页留出空间时，`BufferPoolManagerInstance`也可以将脏页写入磁盘。
为了确保您的实现与系统的其余部分正确配合，我们将为您提供一些已经填写的功能。您也不需要实现实际将数据读写到磁盘的代码（在我们的实现中称为`DiskManager`）。我们将为您提供该功能。
系统中的所有内存页都由页对象表示。`BufferPoolManagerInstance`不需要理解这些页面的内容。但是，作为系统开发人员，了解页面对象只是缓冲池中内存的容器，因此并不特定于唯一页面，这一点很重要。也就是说，每个页面对象都包含一个内存块，`DiskManager`将使用该内存块作为位置来复制从磁盘读取的物理页面的内容。`BufferPoolManagerInstance`将重用相同的页面对象来存储在磁盘上来回移动的数据。这意味着在系统的整个生命周期中，同一页面对象可能包含不同的物理页面。页面对象的标识符（`Page_id`）跟踪它包含的物理页面；如果页面对象不包含物理页面，则其页面id必须设置为无效的页面`id`。
每个页面对象还维护一个计数器，用于“固定”该页面的线程数。不允许您的`BufferPoolManagerInstance`释放已固定的页面。每个页面对象还跟踪它是否脏。您的工作是记录页面在取消固定之前是否已修改。`BufferPoolManagerInstance`必须将脏页的内容写回磁盘，然后才能重用该对象。
您的`BufferPoolManagerInstance`实现将使用您在此分配的前面步骤中创建的`LRUReplacer`类。它将使用`LRUReplacer`跟踪访问页面对象的时间，以便在必须释放一个帧以腾出空间从磁盘复制新的物理页面时，决定退出哪个对象。

需要实现以下函数：

- #### `FetchPgImp(page_id)`

  Fetch the requested page from the buffer pool.

- #### `UnpinPgImp(page_id, is_dirty)`

  Unpin the target page from the buffer pool.

- #### `FlushPgImp(page_id)`

  Flushes the target page to disk.

- #### `NewPgImp(page_id)`

  Creates a new page in the buffer pool.

- #### `DeletePgImp(page_id)`

  Deletes a page from the buffer pool.

- #### `FlushAllPagesImpl()`

  Flushes all the pages in the buffer pool to disk.

下面两个代码段列出了`BufferPoolManagerInstance`中定义的数据成员和构造函数。构造函数分配了pool_size个page，这些page对象（物理页）被重复利用，来存储不同的逻辑页。free_list中存放的是目前空闲（没有逻辑页映射到此页）的物理页页帧，构造函数将pages_ 数组的下标存入free_list中，因为当然开始时物理页都是没被映射的。因为逻辑页的个数会远大于物理页的个数，所以需要用replacer寻找可以替换的物理页，就是把原来的物理页内容写回磁盘，把新的逻辑页的内容从磁盘读出来。逻辑页会在不同的时间映射到不同的物理页，用page_table建立逻辑页页号和物理页页帧的映射。disk_manager_用来根据逻辑页号来存取数据。互斥锁latch用来保证所有的操作都是线程安全的。其他数据成员和task3相关（除了log_manager）。

```c++
/** Number of pages in the buffer pool. */
  const size_t pool_size_;
  /** How many instances are in the parallel BPM (if present, otherwise just 1 BPI) */
  const uint32_t num_instances_ = 1;
  /** Index of this BPI in the parallel BPM (if present, otherwise just 0) */
  const uint32_t instance_index_ = 0;
  /** Each BPI maintains its own counter for page_ids to hand out, must ensure they mod back to its instance_index_ */
  std::atomic<page_id_t> next_page_id_ = instance_index_;

  /** Array of buffer pool pages. */
  Page *pages_;
  /** Pointer to the disk manager. */
  DiskManager *disk_manager_ __attribute__((__unused__));
  /** Pointer to the log manager. */
  LogManager *log_manager_ __attribute__((__unused__));
  /** Page table for keeping track of buffer pool pages. */
  std::unordered_map<page_id_t, frame_id_t> page_table_;
  /** Replacer to find unpinned pages for replacement. */
  Replacer *replacer_;
  /** List of free pages. */
  std::list<frame_id_t> free_list_;
  /** This latch protects shared data structures. We recommend updating this comment to describe what it protects. */
  std::mutex latch_;
```



```c++
BufferPoolManagerInstance::BufferPoolManagerInstance(size_t pool_size, uint32_t num_instances, uint32_t instance_index, DiskManager *disk_manager, LogManager *log_manager)
    : pool_size_(pool_size),
      num_instances_(num_instances),
      instance_index_(instance_index),
      next_page_id_(instance_index),
      disk_manager_(disk_manager),
      log_manager_(log_manager) {
  BUSTUB_ASSERT(num_instances > 0, "If BPI is not part of a pool, then the pool size should just be 1");
  BUSTUB_ASSERT(
      instance_index < num_instances,
      "BPI index cannot be greater than the number of BPIs in the pool. In non-parallel case, index should just be 1.");
  // We allocate a consecutive memory space for the buffer pool.
  pages_ = new Page[pool_size_];
  replacer_ = new LRUReplacer(pool_size);

  // Initially, every page is in the free list.
  for (size_t i = 0; i < pool_size_; ++i) {
    free_list_.emplace_back(static_cast<int>(i));
  }
}
```

#### 获得可用物理页 `GetVictimPage(frame_id_t *frame_id)`

1. 如果bpmi的free_list为空

   -  就使用replacer找到一个最久为使用的pin_conut为0物理页进行替换，脏页要被写回磁盘

   - 如果replacer中没有可以替换的帧，表示所以的物理帧都在被使用，return false

2. free_list不为空，选择第一个物理页帧

```c++
Page *BufferPoolManagerInstance::GetVictimPage(frame_id_t *frame_id) {
    Page *page = nullptr;
    if (free_list_.empty()) {
        bool has_victim = replacer_->Victim(frame_id);
        if (!has_victim) {
            return nullptr;
        }
        page = &pages_[*frame_id];
        page_table_.erase(page->GetPageId());
        if (page->IsDirty()) {
            disk_manager_->WritePage(page->GetPageId(), page->GetData());
            page->is_dirty_ = false;
        }
    } else {
        *frame_id = free_list_.front();
        free_list_.pop_front();
        page = &pages_[*frame_id];
    }
    page->ResetMemory();
    return page;
}
```

####  	

#### `FetchPgImp(page_id)`

此函数功能是根据逻辑页的页号得到页指针

1. 如果page_table中存在页号的键，表示该逻辑页存在于内存中，返回该页对应的物理页即可，需要增加pin_count，此时还需要Pin对应的物理页帧，因为可能存在一种情况。当该逻辑页映射到该物理页而并没有在使用该物理页时（pin_count = 0），该物理页被加入到replacer里，属于可以被牺牲。现在该逻辑页从新使用该物理页时，需要把该物理页从replacer中删除，因为该物理页已经不可被牺牲。
2. 没有该页号的键表示该逻辑页不在内存中，需要把它从磁盘读入到内存。此时就需要获得一个可牺牲的（pin_count=0）的物理页，增加该物理页的pin_count，并把更新数据成员，把该逻辑页的内存从磁盘读入内存（读入到该物理页）。

```c++
Page *BufferPoolManagerInstance::FetchPgImp(page_id_t page_id) {
    // 1.     Search the page table for the requested page (P).
    // 1.1    If P exists, pin it and return it immediately.
    // 1.2    If P does not exist, find a replacement page (R) from either the
    // free list or the replacer.
    //        Note that pages are always found from the free list first.
    // 2.     If R is dirty, write it back to the disk.
    // 3.     Delete R from the page table and insert P.
    // 4.     Update P's metadata, read in the page content from disk, and then
    // return a pointer to P.
    Page *page = nullptr;
    frame_id_t frame_id;
    bool page_exists = false;

    latch_.lock();
    if (page_table_.find(page_id) != page_table_.end()) {
        frame_id = page_table_[page_id];
        page = &pages_[frame_id];
        replacer_->Pin(frame_id);
        page_exists = true;
    } else {
        page = GetVictimPage(&frame_id);
    }
    if (!page) {
        latch_.unlock();
        return nullptr;
    }

    page->page_id_ = page_id;
    page_table_.insert({page_id, frame_id});
    ++page->pin_count_;
    if (!page_exists) {
        disk_manager_->ReadPage(page_id, page->GetData());
    }
    latch_.unlock();
    return page;
}
```

#### `UnpinPgImp(page_id, is_dirty)`

函数功能是当逻辑页使用结束后减少映射的物理页的被引用数。

如果被页面数据被修改过，把物理页属性设为dirty。页面引用数减一，当引用数为0时，该页面可以被牺牲（替换）。

```c++
bool BufferPoolManagerInstance::UnpinPgImp(page_id_t page_id, bool is_dirty) {
    frame_id_t frame_id;
    latch_.lock();
    if (page_table_.find(page_id) == page_table_.end()) {
        latch_.unlock();
        return false;
    }
    frame_id = page_table_[page_id];
    Page *page = &pages_[frame_id];
    page->is_dirty_ = is_dirty || page->is_dirty_;
    if (page->GetPinCount() > 0) {
        --page->pin_count_;
    }
    if (page->GetPinCount() == 0) {
        replacer_->Unpin(frame_id);
    }
    latch_.unlock();
    return true;
}
```

#### `FlushPgImp(page_id_t page_id)`

函数把脏页写会磁盘

```c++
bool BufferPoolManagerInstance::FlushPgImp(page_id_t page_id) {
    // Make sure you call DiskManager::WritePage!
    // disk_manager_->WritePage()
    frame_id_t frame_id;

    latch_.lock();
    if (page_id == INVALID_PAGE_ID ||
        page_table_.find(page_id) == page_table_.end()) {
        latch_.unlock();
        return false;
    }
    frame_id = page_table_[page_id];
    if (pages_[frame_id].IsDirty()) {
        disk_manager_->WritePage(page_id, pages_[frame_id].GetData());
    }
    latch_.unlock();
    return true;
}
```

#### `NewPgImp(page_id_t *page_id)`

函数会生成一个新的逻辑页

- 如果没有可以替换的物理页，pin_count全部大于0，就无法为新的逻辑页分配物理页，所以生成失败
- 有的话，使用`AllocatePage`获得一个新的逻辑页号，然后就是建立逻辑页与物理页的映射，更新页面属性等等

```c++
Page *BufferPoolManagerInstance::NewPgImp(page_id_t *page_id) {
    // 0.   Make sure you call AllocatePage!
    // 1.   If all the pages in the buffer pool are pinned, return nullptr.
    // 2.   Pick a victim page P from either the free list or the replacer.
    // Always pick from the free list first.
    // 3.   Update P's metadata, zero out memory and add P to the page table.
    // 4.   Set the page ID output parameter. Return a pointer to P.
    frame_id_t frame_id;
    Page *page = nullptr;

    latch_.lock();
    page = GetVictimPage(&frame_id);
    if (!page) {
        *page_id = INVALID_PAGE_ID;
        latch_.unlock();
        return nullptr;
    }
    *page_id = AllocatePage();
    page_table_.insert({*page_id, frame_id});
    page->page_id_ = *page_id;
    page->ResetMemory();
    page->pin_count_ = 1;
    latch_.unlock();
    return page;
}
```

#### `DeletePgImp(page_id_t page_id)`

删除该逻辑页的映射，并把对应的物理页加入free_list中。

- 只有当该逻辑页存在映射，才会有对应的物理页，并且对应物理页pin_count等于0才能释放该物理页，显然当别的逻辑页还在使用这个物理页时不能释放这个物理页。
- 删除映射，写回脏页，还原页面成员属性。
- 最重要的一点，还要把这个物理页从replacer中删去，因为可以被删除的页pin_count一定为0，所以这个页一定在replacer中。

**注意**：我之前没有调用`replacer_->Pin(frame_id);`这个函数但同样过了所有的case，但是不调用确实有问题。

如果不调用`pin`，考虑一种情况：

1. 创建一个含有1个物理页的bpm
2. new一个页面然后delete这个页面，这时free_list和replacer里就会存在相同的物理页帧
3. 在new一个page，用到free_list中的物理页，再new一个page时，会从replacer里得到一个页帧。显然出现了错误。

不知道有没有其他人碰到过这个问题，我把测试用例贴在下面，大家可以测试一下自己的实现。

```c++
bool BufferPoolManagerInstance::DeletePgImp(page_id_t page_id) {
  // 0.   Make sure you call DeallocatePage!
  // 1.   Search the page table for the requested page (P).
  // 1.   If P does not exist, return true.
  // 2.   If P exists, but has a non-zero pin-count, return false. Someone is
  // using the page.
  // 3.   Otherwise, P can be deleted. Remove P from the page table, reset its
  // metadata and return it to the free list.
  frame_id_t frame_id;
  latch_.lock();
  if (page_table_.find(page_id) == page_table_.end()) {
    latch_.unlock();
    return true;
  }
  frame_id = page_table_[page_id];
  Page *page = &pages_[frame_id];
  if (page->GetPinCount() > 0) {
    latch_.unlock();
    return false;
  }
  page_table_.erase(page->GetPageId());
  if (page->IsDirty()) {
    disk_manager_->WritePage(page->GetPageId(), page->GetData());
    page->is_dirty_ = false;
  }
  DeallocatePage(page_id);
  
  // pin这个函数需要调用
  replacer_->Pin(frame_id);

  page->page_id_ = INVALID_PAGE_ID;
  free_list_.push_back(frame_id);
  latch_.unlock();
  return true;
}

// 测试代码
TEST(BufferPoolManagerInstanceTest, MyDataTest) {
  const std::string db_name = "test.db";
  const size_t buffer_pool_size = 1;
  auto *disk_manager = new DiskManager(db_name);
  // buffer pool size  = 1
  auto *bpmi = new BufferPoolManagerInstance(buffer_pool_size, disk_manager, nullptr);
  page_id_t page_id_temp;
  // allocate a page
  auto *page0 = bpmi->NewPage(&page_id_temp);
  ASSERT_EQ(0, page_id_temp);
  ASSERT_NE(nullptr, page0);
  // unpin and delete this page
  ASSERT_EQ(true, bpmi->UnpinPage(page_id_temp, false));
  ASSERT_EQ(true, bpmi->DeletePage(page_id_temp));

  // allocate a page
  page0 = bpmi->NewPage(&page_id_temp);
  ASSERT_EQ(1, page_id_temp);
  ASSERT_NE(nullptr, page0);
  
  // 这次分配应该是失败的
  page0 = bpmi->NewPage(&page_id_temp);
  ASSERT_EQ(INVALID_PAGE_ID, page_id_temp);
  ASSERT_EQ(nullptr, page0);

  disk_manager->ShutDown();
  remove("test.db");
  delete bpmi;
  delete disk_manager;
}
```



### 并行缓冲池管理器

这个task非常简单，就是在`parallel_buffer_pool_manager`多分配几个`buffer_pool_manager_instance`,把逻辑页号对`bpmi`的个数取模，调用bpmi数组中下标为取模结果的实例来完成任务即可，提高效率。

