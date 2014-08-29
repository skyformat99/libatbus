# 使用文档

## 编译构建
### 准备工作
+ 支持或部分支持C++11的编译器，至少要支持thread、atomic、智能指针、函数绑定、static_assert (如：GCC 4.4以上、VC 10以上、clang 3.0 以上等)
+ cmake: 2.8.9 以上,如果系统自带cmake版本过低请手动编译安装[cmake](http://cmake.org/)，
+ protocol文件夹内的协议文件是事先生成好的，当然也可以重新生成一份放里面

### 提示
+ GCC 4.4 以上会自动使用-std=gnu++0x, GCC 4.7 以上C++采用 -std=gnu++11， c采用 -std=gnu11。但不会开启C++1y/C++14标准
+ Clang会自动开启到-std=c++11

### 编译选项
除了cmake标准编译选项外，libatbus还提供一些额外选项

+ ATBUS_MACRO_BUSID_TYPE: busid的类型(默认: uint64_t)，建议不要设置成大于64位，否则需要修改protocol目录内的busid类型，并且重新生成协议文件
+ GTEST_ROOT: 使用GTest单元测试框架
+ BOOST_ROOT: 设置Boost库根目录
+ PROJECT_TEST_ENABLE_BOOST_UNIT_TEST: 使用Boost.Test单元测试框架(如果GTEST_ROOT和此项都不设置，则使用内置单元测试框架)


## 开发文档
### 目录结构说明

+ 3rd_party: 外部组件（不一定是依赖项）
+ doc: 文档目录
+ include: 导出lib的包含文件（注意不导出的内部接口后文件会直接在src目录里）
+ project: 工程工具和配置文件集
+ protocol: 协议描述文件目录
+ sample: 使用示例目录，每一个cpp文件都是一个完全独立的例子
+ src: 源文件和内部接口申明目录
+ test: 测试框架及测试用例目录

### 关于 #pragma once
由于目标平台和环境的编译器均已支持 #pragma once 功能，故而所有源代码直接使用这个关键字，以提升编译速度。

详见:[pragma once](http://zh.wikipedia.org/wiki/Pragma_once) 


### 内存通道设计
单多写状态
```
                   ▼       ▼
-------------------WWWW####WWWW###----------------
                   △
```
|长度|           节点头结构           |       说明       |
|---|------------------------------|------------------|
|1B |           标记 flag           |是否写完、是否是头节点|
|1B |        写权限（原子操作）        |       |
|4B |        首读时间（毫秒）          |最大容忍误差是49天|


**通道结构**

所有消息对齐到size_t的大小
|4K通道头|数据节点头*数据节点个数|数据区|

```cpp
// 通道头
typedef struct {
    // 数据节点
    size_t node_size;  /** 每个节点的size **/
    size_t node_size_bin_power; // (用于优化算法) node_size = 1 << node_size_bin_power
    size_t node_count; /** 数据节点个数 **/

    // [atomic_read_cur, atomic_write_cur) 内的数据块都是已使用的数据块
    // atomic_write_cur指向的数据块一定是空块，故而必然有一个node的空洞
    // c11的stdatomic.h在很多编译器不支持并且还有些潜规则(gcc 不能使用-fno-builtin 和 -march=xxx)，故而使用c++版本
    volatile std::atomic<size_t> atomic_read_cur;   // std::atomic也是POD类型
    volatile std::atomic<size_t> atomic_write_cur;  // std::atomic也是POD类型

    // 第一次读到正在写入数据的时间
    uint32_t first_failed_writing_time; /** 第一次读到正在写节点的时间，用于跳过错误写 **/

    volatile std::atomic<uint32_t> atomic_operation_seq; // 操作序列号(用于保证只有一个接收者)

    // 配置
    mem_conf conf;
    size_t area_channel_offset; /** 地址偏移: channel **/
    size_t area_head_offset;    /** 地址偏移: 数据节点头 **/
    size_t area_data_offset;	/** 地址偏移: 数据区 **/
    size_t area_end_offset;		/** 地址偏移: 使用的缓冲区尾部 **/

    // 统计信息
    size_t block_bad_count; 	// 读取到坏块次数
    size_t block_timeout_count; // 读取到写入超时块次数
    size_t node_bad_count; 		// 读取到坏node次数
} mem_channel;

// 配置数据结构
typedef struct {
    size_t protect_node_count;	/** 保护节点个数：用于降低冲突概率 **/
    size_t protect_memory_size;	/** 保护内存大小：用于降低冲突概率 **/
    uint64_t conf_send_timeout_ms;	/** 发送超时阀值：用于降低冲突概率 **/

    // TODO 接收端校验号(用于保证只有一个接收者)
    volatile std::atomic<size_t> atomic_recver_identify;
} mem_conf;
```

**写数据步骤：**

1. 获取写游标，比较读游标，判断是否有空间
2. 分配操作序号
3. 顺序设置操作序号，交换0序号块，失败则返回空间不足
4. 逆序写数据，设置写完状态


**读数据步骤：**

1. 获取读游标，比较写游标，判断是否有数据
2. 分配操作序号
2. 获取第一个节点是否准备完成状态
3. 如果不是完成状态尝试设置首读时间，如果首读超出阀值则认为写错误。此时reset脏节点（非头且不到写游标节点）后移动读游标


**关于冲突：**

1. **读-读冲突：**只考虑单点读，没有这个问题。
2. **读-写冲突：**head有写完毕标记位，当写数据块准备完毕时才开始读。
3. **写-写冲突：**写游标是原子操作，每个节点写缓冲区独立。如果两个节点同时写一个块，则只有一个能写成功。如果写序列中任意块写失败，则整体返回空间不足，写失败。（防止读失败后释放的内存被重新写导致写冲突）
4. **写进程崩溃：**会产生赃数据块，即写完标记永远是未写完。这时候可以利用上上面提到的第一次读取时间。如果是0，则取当前时间赋值，否则如果超出容忍值，就视为赃数据块。取时间可以使用clock函数（Linux下实测每次执行消耗约160ns），也可以用汇编直接提取CPU时钟。一般情况下系统应该在数百次读取无数据后休眠至少一个时间片的时间(Linux下一般最少有4ms)，这时候写进程还没写完基本可以认为是出现赃数据。
5. **读进程崩溃：**移动读游标是最后的操作，下次启动时可以继续，不会丢失数据

**写-读失败-写覆盖问题：**

有一个目前无法解决的是**写-读失败-写覆盖**的问题。这个问题比较难处理，而且发生情况很少。为了这个偶现的问题增加锁和复杂的错误处理逻辑我认为是很不值得的，所以这里采用一些措施来提早发现问题。

+ **第一个措施**是增加一个保护区，当剩于空间不足某个阀值时直接返回空间不足
+ **第二个措施**是增加一个简单的校验码，当校验不通过时返回错误

在设置合理的情况下这两个措施基本能保证数据不出错（如果设置合理，再出错的概率按某人的说法就是，硬件也会出错坏掉的啊）

**共享内存通道压力测试**
1个读进程，5个写进程
读进程满负荷运行3小时，接收数据3390712433次，接收数据12933GB，出现9次数据坏块错误，无数据校验错误
出错率低于3.7亿分之一