# Project1

## 任务

在一个基于磁盘的数据库中实现缓冲池管理器，用于管理内存和磁盘中的数据交换，并实现LRU内存置换策略。

## 概念

- Page：
  - 实际内存数据的封装，即本阶段我们处理的数据单元
  - page_id_t 对应Page在内存的编号
  - Page在内存中组织为数组（page_id不是下标，考虑到多个BPM实例，AllocatePage()）
  - A wrapper for actual data pages being held in main memory
- DiskManager
  - 控制Page数据在内存和磁盘中的读写
  - 其实就是读写文件，bustub中磁盘数据表现为test.db文件
  - 该文件会在 /build/test/ 路径下创建并在测试结束后删除
  - 类似地：LogManager
  - 这一层不需要我们实现
- frame_id_t
  - Page数组的下标；LRU管理的对象
  - Page用数组来组织+LRU管理frame_id链表 **VS** 直接将Page用链表来组织，并移动数据来实现LRU
  - page_id与Page在文件(磁盘)中的位置一一对应，frame_id则是Page在内存Page数组的位置
- LRUReplacer
  - 置换器，基于LRU策略实现，用于决定BPI满时，哪个Page被换出
- BufferPoolManagerInstance
  - 缓存管理器。存储和管理Page。

## 具体实现

先看：测试文件+函数注释

### LRU Replacer

#### 思路

- 使用list组织和存储frame_id
- 使用unordered_map快速定位frame_id
- 对每个“原子操作”，也就是我们实现的函数，进行加锁

#### 私有成员

```c++
// LRUReplacer的排他锁
std::mutex mutex_;

// 存储 frame_id_t
std::list<frame_id_t> lru_list_;

//快速定位
std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> speed_map_;
```

#### 函数

```C++
explicit LRUReplacer(size_t num_pages);
~LRUReplacer() override;
/* 
* 接收一下num_pages,LRUReplacer可存储frame_id的最大数量
* 没有手动申请内存，所以不需要手动析构
*/
```

```C++
bool Victim(frame_id_t *frame_id);
/* 
* 含义：
* 淘汰一个页面，返回被淘汰页面的frame_id，以及操作是否成功
*/

/* 
* 实现：
* 删除list的末尾元素，删除unordered_map的指定元素
*/

```

```C++
void Pin(frame_id_t frame_id);
/* 
* 含义：
* 取出frame_id指定的页面，意味着该页面需要留在内存中，不能被换出
*/

/* 
* 实现：
* 删除list以及unordered_map中的指定元素
*/
```

```C++
void Unpin(frame_id_t frame_id);
/* 
* 含义：
* 插入frame_id指定的页面，意味该页面可以被换出内存
*/

/* 
* 实现：
* 向list以及unordered_map中插入指定元素
* 注意：基于LRU策略，新元素需要插入到list的开头位置
*/
```

```C++
size_t Size();
/* 
* 含义：
* 返回当前LRU中管理的页面数量
*/

/* 
* 实现：
* 返回链表元素的数量
*/
```

### Buffer Pool

#### 思路

- Page数组的划分
  - 在free list，尚未被使用/删除后释放
  - 不在free list，占用中
    - 在LRU，不可换出
    - 不在LRU，可换出

- Page table：所有内存中占用的Page的page_id与frame_id的映射
  - frame可以分为：在page_table和在free_list两部分

- 数据
  - 在内存，可通过page id在page table中查到frame id
  - 在磁盘，BP中查不到信息，猜测BP上层会存储所有有效的page id，以控制删除后，不能传参无效page id调用Fetch

- Pin()：被占用的Page不能换出

#### 私有成员

```C++
// Page数组
Page *pages_;

// Page table for keeping track of buffer pool pages. 
// 存储page_id到frame_id的映射
std::unordered_map<page_id_t, frame_id_t> page_table_;

// Replacer to find unpinned pages for replacement
Replacer *replacer_;

// List of free pages，代表尚未使用的Page
std::list<frame_id_t> free_list_;

// 对原子操作，也就是每个函数，加锁
std::mutex latch_;
```

#### 函数

```C++
BufferPoolManagerInstance(size_t pool_size, uint32_t num_instances, uint32_t instance_index, DiskManager *disk_manager, LogManager *log_manager)
/* 
* 实现：
* 接收元信息
* 初始化磁盘/日志控制器，LRU置换控制器
* 根据pool_size，手动申请内存
*/
```

```C++
~BufferPoolManagerInstance()
/* 
* 实现：
* 释放所有的Page，释放LRU置换控制器
*/
```

```C++
Page* NewPgImp(page_id_t *page_id);
/* 
* 含义：
* 在BP中申请一个可用Page，返回它的page_id，和指向它的指针
*/

/* 
* 实现：
* 寻找可用页面（封装为单独的函数：FindFreshPage）
    * 先在free list，寻找空页
    * 如果找不到，则去LRU中进行Victim，注意对脏页刷盘
    * 如果LRU中都被Pin，则返回nullptr
    * 如果找到可用页，则重置可用页
* 初始化页面
	* 将页面从LRU中取出，代表有一个线程正在使用
	* 分配page_id
	* 修改页面元数据，修改page_table_
	* 将页面刷盘
* return
*/
```

```C++
Page* FetchPgImp(page_id_t page_id);
/* 
* 含义：
* 根据指定page_id，将对应Page取到内存，并返回指针
*/

/* 
* 实现：
* 在内存寻找这个页面（封装为单独的函数：FindPage）
    * 如果找到，将页面从LRU中取出，代表有一个线程正在使用
    * 修改页面元数据
    * return
* 申请一个可用页面，用于将这个页面的数据从磁盘读到内存
	* 类似NewPgImp()
	* 只不过不用分配page_id，且调用disk_manager_->ReadPage，数据流动方向改变
	* return
*/
```

```C++
bool FlushPgImp(page_id_t page_id);
/* 
* 含义：
* 将指定page_id对应的Page，进行刷盘
*/

/* 
* 实现：
* 通过page_table_[page_id]，找打frame_id，进而找打Page数据
* disk_manager_ 刷盘
* 重置脏位
*/
```

```C++
void FlushAllPgsImp();
/* 
* 含义：
* 将所有Page，进行刷盘
*/

/* 
* 实现：
* 注意只能加一个锁
* FlushPgImp实现为了加锁版本的FlushPg，封装为单独的函数
* 因此，FlushAllPgsImp应该加锁后调用FlushPg，而不是FlushPgImp
*/
```

```C++
bool UnpinPgImp(page_id_t page_id, bool is_dirty);
/* 
* 含义：
* 根据指定page_id，释放对指定Page的占用
*/

/* 
* 实现：
* 在内存寻找这个页面（封装为单独的函数：FindPage）
* 如果找不到，或者没人占用这个Page，return
* 否则
	* 若is_dirty==true，修改脏位
	* 修改Page占用计数
	* 如果修改后占用为0，则将Page放回LRU，代表无人占用，可以换出
*/
```

```C++
bool DeletePgImp(page_id_t page_id);
/* 
* 含义：
* 根据指定page_id，删除指定Page
*/

/* 
* 实现：
* 在内存寻找这个页面（封装为单独的函数：FindPage）
* 如果找不到，return
* 如果找到
	* 若GetPinCount！=0， return
	* 否则
        * 将该页面从LRU中取出，代表删除
        * 重置页面
        * 修改page_table_，修改free_list_
        * 销毁page_id
        * return
*/
```

## 逻辑问题

### LRU查找

- 以存储换速度
- 如果LRUReplacer中没有map结构，而是通过便利查找frame id的位置，或判断是否存在，则耗时太多，线上测试会超时

### 刷盘时机

- FlushAllPgsImp()：无论是否dirty都必须刷盘
  - 因为可能某个线程修改了Page，但是还没有释放对Page的占用（Unpin），也就意味着此时Page没有置为dirty，所以所有Page都要刷盘
- NewPgImp()：先在内存申请Page，再刷到磁盘
- Victim()：驱逐页面非脏不能刷盘

### UnpinPgImp

- 这个函数的is_dirty参数，代表这个线程是否修改了Page
- 若is_dirty参数为false，则Page的dirty保持原样（原来可能为T或F）
- 若is_dirty参数为true，则Page的dirty可能要置为T

### DeletePgImp

- 如果内存中找不到这个Page，就直接返回true
- 这意味着，若Page在磁盘，不去管他
- 应该是一种简单实现：磁盘空间用了就不需要释放，无法找到这块空间就代表删除了。DiskManager没有delete
- 猜测：所有new过的page_id，应该是由buffer pool上层持有，并且以此控制Delete之后，page_id变为无效，上层也不会Fetch无效id

### FetchPgImp

- 同DeletePgImp，我们认为上层传入的page id都是有效的
- 所以这个Page，不是在内存就是在磁盘

### 有效page id

- 其实可以在BPIM中存储有效page id的集合
- 在调用函数时，首先验证page id的有效性

## 残留问题

- 为什么暴力遍历LRU链表会超时？
- 为什么驱逐页面时，对非脏页也刷盘无法通过线上测试（猜测判断了刷盘次数）？
- FlushAllPgsImp(); 是否是无论页面是否为脏，都要刷盘？



