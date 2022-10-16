不光要实现每个填空，还要理解整个bustub的架构，tinykv，polar也是这样

这个BP，与polar的区别；这两个demo代码分别往课程的章节上对应呢？

## 问题记录

### 1

```
问题：make format：permision denied

解决方法：sudo chmod -R 777 build_support/*
```

### 2

想要测试内存泄漏，将cmake中的mode修改为release

toolchain中的C++ 编译器，使用空白、clang-12、clang-10都编译不过，而且是项目原有文件有错误

比如项目1线上测试显示内存安全不通过，但是debug模式下，本地三个测试用例都通过了valgrind测试

### 3

- P1：dirty修改规则；flush规则；偶尔的内存检测超时
- P2多次fetch与unpin的对应

```C++
100%，内存安全检测超时

// 放回
void LRUReplacer::Unpin(frame_id_t frame_id) {
  std::lock_guard<std::mutex> lock(mutex_);
  // 如果该元素已经存在与LRU，则什么都不做
  if (!IsInReplacer(frame_id)) {
    lru_list_.emplace_front(frame_id);
    speed_map_.insert({frame_id, lru_list_.begin()});
  }
}

bool LRUReplacer::IsInReplacer(frame_id_t frame_id) {
  std::list<frame_id_t>::iterator iter;
  bool re = false;
  for (iter = lru_list_.begin(); iter != lru_list_.end(); iter++) {
    if (*iter == frame_id) {
      re = true;
      break;
    }
  }
  return re;
}
```

## 环境搭建

1. 项目第一次打开（注意设置settings->build->cmake，并管理工具链），根目录存在cmakelists.txt就会自动load cmake project吗？
2. load cmake project之后，clion识别到了所有的CMake Application和Google Test
3. clion如何识别所有的CMake Application，这些CMake Application并不能在cmakelists.txt中全部找到？这些Application有啥用？
4. clion如何识别所有的Google Test

## 线上测试、google格式检测、日志等级管理

## 内存泄漏的避免和线程安全的保证

## lab0

### 要求

- Fill in the implementations of all the constructors, destructors, and member functions.
- Do not add any additional function prototypes or member variables.

## lab1

### 要求

- You **should not** modify the signatures for the pre-defined functions in these classes.
- You also **should not** add additional classes in the source code for these components. 
- If a class already contains data members, you should **not** remove them. 
- you may need to add data members to these classes.
- You can also add additional helper functions to these classes
- You are allowed to use any built-in [C++17 containers](http://en.cppreference.com/w/cpp/container).（注意额外实现线程安全）

### 理解

- task2：内存中所有的frame，分为三部分
  1. free_list_：完全没有数据，对应清空的frame
  2. lru：有数据，但是没有人pin，可以被换出
  3. 不在上面两个结构中：有数据，pin不为0，不可被换出（page_id不为invaild也算有数据）
- task2：page_table_包含了所有在内存的page的page_id与frame_id的映射
  - frame还可以分为，在page_table和在free_list两部分
- task2：BufferPoolManagerInstance::NewPgImp需要Pin()
  - 从测试代码可以看出，NewPage占用所有page后，不希望再有新的page可以被new，也就意味着这时候free_list和lru里面都应该为空，所以刚new完的page应该pin_count=1
  - 同理，NewPgImp()和FetchPgImp()的非空指针返回结果，应该即不在free_list_中，也不在lru中
- task2：
  - 为什么一旦fetchpage，is_dirty就要置为true呢？？？
  - unpin到为0的时候，用不用FlushPg(page_id);？？？

- task3：ParallelBufferPoolManager::NewPgImp可以说是按照官网文档的要求来做的，但是这种做法的优点在哪呢？
- 其他：内存泄漏的原因？？？

## lab2

### 理解

- 到底是bp调用hb，还是hb调用bp？？？这个hb管理的是内存？BP的理解还是不到位
- task1:
  - bucket page内部无序，因为是顺序插入的
  - 理解occupied和readable的具体含义
  - 掌握GetMask()、SetOccupied()、IsOccupied()、NumReadable()中涉及到的位操作，及其含义
- task2:
  - 两类hash_table_page的大小和bp中的page的data区大小相同
    - BP page，GetData()前后的区别？？？
  - 很重要
    - 注意UnpinPage()的时机
    - merge的规则，merge后是否需要 delete bp page？？？
    - split和merge时，考虑多个bucket_idx指向同一个bucket的情况
  - 使用 **least-significant bits** 真的会让代码实现相对简单
  - 编程时，区分bucket_idx和bucket_page__(data)_id;
  - GetSplitImageIndex()的含义与实现？？？
- task3:
  - table_latch_是 hash table上的锁，也就是控制directory page的并发读取
    - // Readers includes inserts and removes, writers are splits and merges
    - 对于table_latch_，因为insert和remove不会改变dir_page，所以这两个操作只需要申请RLock
    - 但是split和merge，会涉及到dir_page的扩张或者收缩，以及映射关系的变动，需要WLock
  - bucket page则需要单独的 page 锁；注意这个锁是GetData()前的锁
  - Unlatch()一定要在UnpinPage之后吗？为什么buffer pool直接在单元操作处加锁，但是bucket page需要在上层调用时，对基于bucket page的一组操作进行加锁？

## lab3

### 理解

- 总结自定义结果体的unordered_map用法
- 总结从tuple中取值的两种用法及细节（distinct，aggre）
- 总结语法问题




## 语法问题

### lab0

注意纯虚函数的定义

```
this->linear_ = new T[rows_ * cols_];
this->data_ = new T *[this->rows_ * this->cols_];
为什么一个加 *，一个不加
T *代表二维？
对应的delete的写法？

pages_ = new Page[pool_size_];
replacer_ = new LRUReplacer(pool_size);
```

```
RowMatrix(int rows, int cols) : Matrix<T>(rows, cols) 
这里RowMatrix的row_,col_，是否已经自动接受了Matrix<T>(rows, cols)的rows, cols？
```

```
virtual auto GetRowCount() const -> int = 0;		//虚函数
auto GetRowCount() const -> int override { return this->rows_; }//实现
为什么要这么写
```

```
if (static_cast<int>(source.size()) != this->rows_ * this->cols_) 
static_cast<int>的作用：类型转换
```

```
static auto Add(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB) -> std::unique_ptr<RowMatrix<T>> 
分析这一行，比如模板类型，形参加const，返回值类型，->这个箭头等
```

```
auto result = std::make_unique<RowMatrix<T>>(matrixA->GetRowCount(), matrixA->GetColumnCount());
make_unique什么操作
unique_ptr干啥的？？？为什么这里返回值是这种指针？

Matrix<T> *re = new Matrix<T>{matrixA->GetRowCount(), matrixA->GetColumnCount()};这种表达怎样才能对？
```

```
子函数访问父类的私有成员变量是要加上this->，访问自己的私有成员变量，可以不加
```

### lab2

```
auto bucket_page = reinterpret_cast<HashTableBucketPage<int, int, IntComparator> *>(
      bpm->NewPage(&bucket_page_id, nullptr)->GetData());
什么意思

reinterpret_cast<HashTableDirectoryPage *>(buffer_pool_manager_->FetchPage(directory_page_id_)->GetData());
```

```
BUCKET_ARRAY_SIZE怎么算出来的
```

```
代码里面用char来做位图的修改和保存的做法，是一种标准模式
```

```
MappingType array_初始化的结果是什么？
如果在HASH_TABLE_BUCKET_TYPE::GetValue和HASH_TABLE_BUCKET_TYPE::Remove中，不对IsOccupied做限制的话，会出现空kv对和函数参数比较的情况，但是不会报错
```

```
if (bucket_idx >= BUCKET_ARRAY_SIZE) return KeyType{};
if (bucket_idx >= BUCKET_ARRAY_SIZE) return {};
写法是否标准？
不标准，因为KeyType取int时，KeyType{}和int{0}相等，建议返回值类型为pair<KeyType,bool>

{}与 KeyType{};一样吗？
```

```
explicit为什么构造函数一般都添加这个关键字

内联函数又有什么效果
 private:
  inline auto Hash(KeyType key) -> uint32_t;
```

```
auto HashTableDirectoryPage::Pow(uint32_t base, uint32_t power) const -> uint32_t {
  return static_cast<uint32_t>(std::pow(static_cast<long double>(base), static_cast<long double>(power)));
}
目的？
```

```
HASH_TABLE_TYPE::KeyToDirectoryIndex为什么要这么实现
```

```
  auto bucket_page_data = reinterpret_cast<HASH_TABLE_BUCKET_TYPE *>(bucket_page->GetData());
  auto bucket_page_data1 = reinterpret_cast<HashTableBucketPage *>(bucket_page->GetData());
  上对，下错
```

```
GetSplitImageIndex的实现
```

```
P2: imple hb get&insert&split commit中，HASH_TABLE_TYPE::SplitInsert函数中：
auto global_depth = dir_page_data->GetGlobalDepth();
auto global_depth = GetGlobalDepth();
为什么下面这个测试无法通过？？？
```

