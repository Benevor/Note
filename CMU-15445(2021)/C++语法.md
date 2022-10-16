- 非列表初始化，到底有没有调用无参数构造函数？？？wsl::code::chushihua代码哪里错了？
- 传指针：如果调用的时候没有生成对象， 传入函数的不是对象的地址而是该类型的空指针，那么在函数内部，能不能用指针指向函数内局部变量？局部变量空间会不会被回收？

## 关键字

### explicit？

- 防止只有一个参数的类构造函数的隐式自动转换
- https://juejin.cn/post/6995340162574073893

### const

- 声明变量仅仅可读，不能修改

  ```C++
  for (const auto &page : page_table_) {}
  testDemo(const testDemo& other) :num(other.num) {}
  ```

### static

- 静态成员不能用花括号初始化？

### atomic

## 特殊函数

### 构造函数？

- 构造函数/拷贝构造函数/移动构造函数
- 为什么拷贝构造函数一般使用const testDemo& other
- 类对象向类对象的赋值都是拷贝构造函数吗
- 深拷贝、浅拷贝咋区分
- 拷贝构造函数和等号操作符的重载有啥区别
- 移动构造函数又是啥
- https://www.cnblogs.com/foghorn/p/15882774.html
- https://blog.csdn.net/weixin_41675170/article/details/93485655

### 析构函数

## STL容器

### emplace_back()和push_back()

- 底层实现机制不同
- https://www.cnblogs.com/foghorn/p/15882774.html

## 编译

### unused

```C++
__attribute__((__unused__))
```

- 可能不会用到，消除编译警告
- https://blog.csdn.net/Rong_Toa/article/details/87926187

## 其他

### (类)变量初始化？

变量初始化

- 列表初始化/拷贝初始化/直接初始化......
- https://blog.csdn.net/q1302182594/article/details/47423347
- 更推荐下面这一篇
- https://www.cnblogs.com/boydfd/p/4981504.html

构造函数类对象初始化

- 类成员变量初始化顺序：声明初始化、列表初始化、构造函数初始化
- https://blog.csdn.net/xihuanzhi1854/article/details/104550834
- 类成员初始化列表：效率更好，避免先初始化，再赋值
- 因为C++在**进入构造函数的函数体之前就完成了类对象的初始化**，所以成员初始化列表的方式就是直接对成员初始化，而在构造函数内部修改成员的方式，其实是先初始化为默认值，再进行赋值修改。

### 函数传值/传引用

- 两者区别：形参与实参的关系
- 通过调用函数改变实参的值：传引用 / 传指针
- https://www.cnblogs.com/zjutzz/p/6818799.html

#### 传指针

- 传指针其实是传值，只不过这个值是指针
- 调用函数时，传指针作为参数，虽然是传值，仍旧可以改变函数外的值

```C++
// 示例
auto LRUReplacer::Victim(frame_id_t *frame_id) -> bool {
	*frame_id = wait_list_.back();
    return true;
}

// 调用
frame_id_t frame_id = -1;
bool res = replacer_->Victim(&frame_id);

// 分析
调用Victim()，传入的是函数外frame_id的地址，在Victim内部的赋值操作，其实是对函数外frame_id的拷贝赋值。
```

### 内存申请与回收

- C++内存空间：自由存储区可以认为是堆
- https://blog.csdn.net/qq_35451572/article/details/83302054
- new和delete的使用
- https://www.cnblogs.com/lca1826/p/6506183.html

### 左值/右值

### 类对象大小

- 假设一个类Page是一个数据区的类抽象，其成员变量包含了

  ```C++
  char data_[PAGE_SIZE]{};
  ```

  以及其他的一些元数据

- 那么这时候，就要区分，PAGE_SIZE != sizeof(Page)

## 问题

### 拷贝构造函数

```C++
// buffer_pool_manager_instance.cpp
Page &delete_page = pages_[frame_id];	// √
Page victim_page = pages_[frame_id];	// ×
下面这个是编译器自动生成拷贝构造函数失败了？
    上面不用调用拷贝构造函数？
Page对象里面含有const成员变量行不行？
// 报错：copy constructor of 'Page' is implicitly deleted because field 'rwlatch_' has a deleted copy constructor 'ReaderWriterLatch' has been explicitly marked deleted here
```

### 构造函数非列表初始化

- 到底有没有调用无参数构造函数？
- wsl::code::chushihua代码哪里错了？

### 函数传指针

- 如果调用的时候没有生成对象， 传入函数的不是对象的地址而是该类型的空指针，那么在函数内部，能不能用指针指向函数内局部变量？局部变量空间会不会被回收？

### 还没看的

- https://www.cnblogs.com/boydfd/p/4981504.html
- https://blog.csdn.net/weixin_41675170/article/details/93485655