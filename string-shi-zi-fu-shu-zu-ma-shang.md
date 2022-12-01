# String是字符数组吗-上

## 一、从一次性能优化说起

> 声明 ：故事纯属虚构，旨在从实际问题切入到文章主题，请不要过分讨论故事内容和技术细节。

### 1、项目背景

某中型互联网公司开拓了一个在线商城的新业务，计划一个多月后赶在双11前上线，并由小王担任技术负责人。在成立之初，小王便规划了整个业务的技术架构，其中由小程负责用户系统的设计。

### 2、用户系统的设计

这是小程第一次负责这么大的业务模块设计，在他受命之际，为了不负厚望，开始认真的规划起用户系统的设计。

### 3、缓存方案的选择

在设计方案即将完成时，为了保证读性能，小程又在原有设计中增加了缓存层。经过多次考虑，他选择了如下的Redis缓存方案。

| Key名称          | Key类型 | 值                                                                     |
| -------------- | ----- | --------------------------------------------------------------------- |
| user:\{{uid\}} | Hash  | <p>uid:12837932<br>name:"Lucky"<br>avatar:"cdn.img.com/edadf.png"</p> |

### 4、项目开发和测试

在小程最终确认设计方案后，开始投入到没日没夜的紧急开发中，虽然累，但也觉得非常值得。在一个多月后，经历了多次测试和验证的用户系统，顺利进入了上线阶段。

### 5、紧张的时刻到了

新业务终于在双11前一晚如期上线了。这天大家都在紧张看着监控，生怕系统出现故障，可是不幸的事情还是在意料之中发生了。

运维告知小程，从晚上10点开始，告警群中开始出现用户服务的Redis网络流量告警，短短的5分钟内总流量异常增长了10倍。小程立即打开监控，发现预热的这段时间内，用户数确实翻了几倍，但远没有10倍，现在问题来了，流量的元凶在哪里 ？

### 6、快速排查与修复

既然问题出现在Redis层，小程便开始从Redis层认真梳理和思考起来 ：在底层获取用户信息的时候，使用了`hgetall`的方式，如果同时段调用`hgetall`的次数过多，再加上有些极个别用户的字段数据较多，那么可能就会导致Redis网络IO的飙升。

小程立即开始排查Redis慢日志，从慢日志中发现有一条看似元凶的命令`hgetall user:18765432`耗时达到了10秒以上，通过分析发现`user:18765432`这个Key竟然达到了惊人的`100MB+`，其中有一个`intro`的个人介绍字段占了99%的空间。这时小程才突然晃过神来，这个字段可能有输入漏洞，导致用户绕过限制恶意上传了大量文本内容导致这个Key异常的大。

来不及排查具体原因，小程先删除了这个用户Key的`intro`字段，然后在后端添加了更严格的校验，堵住用户侧的漏洞，等了大概1分钟后系统终于恢复了正常。

### 7、重新思考和设计

为了系统地优化Redis性能，小程重新设计了 `zip + protobuf` 的缓存方案。这种方案是否可行呢，下面会进行测试实验。

* `zip` ：对用户的长文本字段进行zip压缩
* `protobuf` ：对用户整体结构做protobuf编码

## 二、场景复现

### 1、实验内容

首先定义一个`Intro`字段长度为800w长文本数据的用户，然后使用`hash`/`json`/`zip+protobuf`三种用户信息存储方式，分别测试数据是否可以正常写入解析和Redis的内存占用情况。

#### (1) 测试代码

经过以下运行发现三种方式数据均可正常写入和解析，由于篇幅限制，完整代码请见[shark项目](https://github.com/WGrape/bookcode/blob/main/shark/main.go)。

```go
// 通过使用三种不同的方式, 观察redis内存的变化

// hash处理方式
hashstore.WriteRedis(redisConn, user)
redisUser = hashstore.ReadRedis(redisConn, user.Uid)
fmt.Printf("%d : name=%s , address=%s , intro.length=%d\n", redisUser.Uid, redisUser.Nick, redisUser.Address, len([]rune(redisUser.Intro)))
// 12345678 : name=用户昵称 , address=用户地址 , intro.length=8000000

// JSON处理方式
jsonstore.WriteRedis(redisConn, user)
redisUser = jsonstore.ReadRedis(redisConn, user.Uid)
fmt.Printf("%d : name=%s , address=%s , intro.length=%d\n", redisUser.Uid, redisUser.Nick, redisUser.Address, len([]rune(redisUser.Intro)))
// 12345678 : name=用户昵称 , address=用户地址 , intro.length=8000000

// Zip+Protobuf处理方式
zipstore.WriteRedis(redisConn, user)
redisUser = zipstore.ReadRedis(redisConn, user.Uid)
fmt.Printf("%d : name=%s , address=%s , intro.length=%d", redisUser.Uid, redisUser.Nick, redisUser.Address, len([]rune(redisUser.Intro)))
// 12345678 : name=用户昵称 , address=用户地址 , intro.length=8000000
```

#### (2) 实验前Redis内存

在实验前，本地Redis为空，默认的空间大小为`used_memory_human:1.21M`

![image](https://user-images.githubusercontent.com/35942268/199239364-799c064f-0773-4951-8f84-f4244e352e50.png)

#### (3) 实验后Redis内存

如果使用Hash存储方式，占用内存为 `24.22 - 1.21 = 23.01MB` ![image](https://user-images.githubusercontent.com/35942268/199247354-ed8b86bc-4c00-4e40-93b7-d2b0bd9b0c41.png)

如果使用Json存储方式，占用内存为 `24.26 - 1.21 = 23.05MB` ![image](https://user-images.githubusercontent.com/35942268/199247772-0fe2a435-833f-465d-8138-721226aa492d.png)

如果使用zip+protobuf存储方式，占用内存为 `1.43 - 1.21 = 0.22MB` ![image](https://user-images.githubusercontent.com/35942268/199248127-113bb8f8-3dc8-4bc5-ba1c-82166fcb9c58.png)

### 2、测试结论

| 测试序号 | 数据量级                   | 方案           | 类型     | 内存      |
| ---- | ---------------------- | ------------ | ------ | ------- |
| 1    | 长度为800w的字符串/800w字节/8MB | Hash存储       | Hash   | 23.01MB |
| 2    | 长度为800w的字符串/800w字节/8MB | JSON存储       | String | 23.05MB |
| 3    | 长度为800w的字符串/800w字节/8MB | Zip+Protobuf | String | 0.22MB  |

经过测试可以得出结论 ：在Redis的String中存储压缩后的字节流数据，不但可以正常解析，还明显节省了大量的内存空间

### 3、聊聊String与Byte\[]

#### (1) 两种类型的差异

* Byte\[]存储最底层的二进制数据，又称为字节流数据，二进制数据
* String是比Byte\[]更高级的类型，是一种在Byte\[]之上抽象出来的类型

因为String和Byte\[]在最底层的存储是一样的，都是二进制数据，所以在Go语言中我们经常看到String和Byte\[]互相转换的情况。

```go
var bytes = []byte("hello world")

// [104 101 108 108 111 32 119 111 114 108 100]
fmt.Printf("%v", bytes)

// hello world
fmt.Printf("%s", bytes)
```

#### (2) 两种数据的写入

由于String与Byte\[]在底层存储并无差异，所以在上面的测试代码中也能发现无论写入String数据还是字节流数据，都能正常写入和读取解析。

也就是说，在Redis中String类型对字符串和二进制两种编码都是支持的，那么它的底层是如何实现的呢 ？我们先留一个悬念。

### 4、C语言中的String

#### (1) 编译期固定长度

在C语言中是没有字符串`string`类型的，一般只有使用字符数组`char[]`才能实现字符串类型，如下三种实现字符串的方式所示。虽然后面两种没有在定义时声明长度，但和第一种`char [N]`是等效的，因为它们的长度都是在编译时就已经固定的。

```c
// 本质都是字符数组
char s[12] = {'h','e','l','l','o',' ','w','o','r','l','d','\0'};
char *s = "hello world";
char s[] = "hello world";
```

#### (2) 运行期动态扩容

有没有一种不固定长度的字符串实现方法呢？当然！如下所示的这两种写法都是定义了`char *`字符指针，然后在运行期间使用如`malloc()/memcpy()`等内存操作函数动态地扩容。

```c
// 本质都是字符指针
char *s;
char s[];
```

### 5、Redis中的String

我们都知道Redis是使用C语言编写的，现在问题来了，Redis中的String是字符数组实现的吗 ？肯定不是！Redis不可能会在编译期间就确定所有字符串的长度，所以只能使用字符指针在运行期动态扩容的方式。

但仅仅是字符指针吗？只要我们简单翻看Redis源码，就会知道它在字符指针`char[]`之上又封装了一种叫做`SDS`的结构。这样实现的原因是什么呢 ？

## 三、String为什么要这样设计

### 1、二进制安全

在C语言中要求字符串末尾必须有`\0`字符（对应的二进制为`0000 0000`），在操作字符串时遇到`\0`字符才会认为字符串结束，这样就会存在以下问题

* 字符串中不能有`\0`这样的二进制数据
* 在执行`strcpy()`、`strcat()`、`memcpy()`操作时有缓冲区溢出的风险

所以在C语言中的字符串不是二进制安全的，而Redis的String为了实现图片、音频等各种二进制数据的存储，就必须解决二进制存储问题，实现二进制安全。

### 2、高性能操作

同样的，由于`\0`字符的限制，在C语言中处理字符串时也会比较严重的性能问题

* 较频繁的内存分配
* 计算字符串长度的时间复杂度为O(N)

因此在底层`SDS`结构中分别定义了`char buf[]`和`len`这两个属性，可以在O(1)常数时间内获取字符串的长度，而且在字符串变更操作时，对`buf[]`数组也设计了更加高效的内存分配策略，提高字符串的操作性能。

## 四、从源码探究String的设计

### 1、SDS

在`src/sds.h`文件中定义了SDS（Simple Dynamic Strings Header）结构和操作API，可以发现SDS并非只有1种，而是定义了`sdshdr5`、`sdshdr8`、`sdshdr16`、`sdshdr32`、`sdshdr64`这五种结构。

![image](https://user-images.githubusercontent.com/35942268/200104615-df86b98b-9360-4f9d-9823-c2754645732b.png)

从这5种结构可以看出，Redis对`len`和`alloc`这些属性都细化了`uint8_t/uint16_t/uint32_t/uint64_t`这4种类型，而不是直接用`int`代替。为了节省内存和提高性能，Redis可谓是 “无所不用其极“ ！

#### (1) 结构定义

* `buf` ：缓存数组，存储实际的字符串
* `len` ：当前字符串的长度，即`buf`已经使用的长度
* `alloc` ：buf 分配的长度，等于buf\[]的总长度-1，因为buf有包括一个/0的结束符
* `flags` ：只有3位有效位，因为类型的表示就是0到4，所有这个8位的flags 有5位没有被用到

#### (2) 3种编码

在`/src/server.h`中定义了不同object（如val值）的编码，其中对String一共有3种编码，具体在后面会详细介绍。

```c
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
```

#### (3) 常用API

* `void sdsfree(sds s);` ：释放一个SDS字符串
* `sds sdsnewlen(const void *init, size_t initlen);` ：创建一个固定长度的新的SDS字符串
* `sds sdscpylen(sds s, const char *t, size_t len);` ：将固定长度的字符串t复制到字符串s中
* `sds sdscatlen(sds s, const void *t, size_t len);` ：将固定长度的字符串t附加到字符串s后面
* `sds sdsgrowzero(sds s, size_t len);` ：将SDS增长到指定的长度，然后将不属于字符串的字节部分都置为0

### 2、Set操作过程

#### (1) 找到Set命令

在`/src/commands.c`文件中的`redisCommandTable[]`数组中定义了set命令的详细内容

![image](https://user-images.githubusercontent.com/35942268/200109520-172844cc-942e-4956-9b88-b4a63d9695b1.png)

其中就包括了`setCommand()`函数，它定义在`t_string.c`文件中，主要包括以下三个操作。

* 解析String类型的`SET`和`GET`命令的扩展参数部分，如`EX/PX/NX`等参数值
* 调用`tryObjectEncoding`函数对Val进行编码，确认Val是`RAW/INT/EMBSTR`3种编码中的哪一个类型
* 调用`setGenericCommand`函数执行SET的核心逻辑

![image](https://user-images.githubusercontent.com/35942268/200124315-ba4fbf06-cab3-4273-8153-ce34bd380b71.png)

#### (2) 生成毫秒时间戳

在`setGenericCommand()`核心函数中先调用`getExpireMillisecondsOrReply()`函数生成毫秒时间戳`milliseconds`

![image](https://user-images.githubusercontent.com/35942268/200124395-56f993e6-3745-48d4-a708-5edcde4ee60e.png)

它是根据`EX/PX`的不同生成以毫秒为单位的时间，与`commandTimeSnapshot()`获取的当前毫秒时间戳相加而成。

![image](https://user-images.githubusercontent.com/35942268/200125181-260dfb7f-1465-4e41-a1c8-32f6fb92b733.png)

#### (3) 如果是GETSET命令则先调用GET

根据`flags`标志判断是否要在SET前先GET，比如`GETSET`命令，如果是则先调用`getGenericCommand()`函数即`GET`命令的核心逻辑，用于返回数据给客户端。

![image](https://user-images.githubusercontent.com/35942268/200125789-c2534230-7f00-4466-b327-d06a421ccc6d.png) ![image](https://user-images.githubusercontent.com/35942268/200126125-ee92c6ab-82f9-4f39-bb86-37740f2bb221.png)

#### (4) 有NX或XX选项的前置处理

在SET命令中，它支持`NX/XX`两种选项

* `NX` ：只在键不存在时，才对键进行设置操作。`SET key value NX` 效果等同于 `SETNX key value`
* `XX` ：只在键已经存在时，才对键进行设置操作

![image](https://user-images.githubusercontent.com/35942268/200129966-b7218882-139b-4361-ad6e-e1a168ba2d0b.png)

在实现中，会先调用`lookupKeyWrite()`函数判断Key是否存在，如果满足下面两种情况，则会执行提前返回的前置处理

* 有`NX`选项，但Key已存在
* 有`XX`选项，但Key不存在

![image](https://user-images.githubusercontent.com/35942268/200129705-9853463b-d6d2-43b7-920c-66bdcb5fc2a3.png)

#### (5) 写入KV值和设置有效期

在调用`setKey()`写入KV前，先根据Key是否存在、是否设置有效期等特点写入`setkey_flags`标志变量中，完成数据的写入，再调用`setExpire()`函数设置有效期。

![image](https://user-images.githubusercontent.com/35942268/200131040-f836b968-7208-497a-b1da-82ddc2bda229.png)

### 3、Get操作过程

同样的，我们在`src/commands.c`文件中可以找到`GET`命令对应的`getCommand()`函数

![image](https://user-images.githubusercontent.com/35942268/200131903-df1052aa-47a7-4207-9c26-9b0575cc9c6a.png)

在`getCommand()`函数中再调用核心函数`getGenericCommand()`，通过`lookupKey()`找到字典中Key对应的Val。

![image](https://user-images.githubusercontent.com/35942268/200132030-156a7604-5191-437f-8e46-3b8ed9c7bc63.png)

在返回Val至客户端前，先判断Val的编码类型，如果是`OBJ_ENCODING_RAW`或`OBJ_ENCODING_EMBSTR`类型，则根据原编码返回数据，如果是`OBJ_ENCODING_INT`类型则转为字符串类型后返回。

![image](https://user-images.githubusercontent.com/35942268/200161872-5f170738-f014-49b4-ac6c-353e12491e72.png)

## 五、未完待续

* `SDS`在源码中的使用？
* `SDS`在内存中的分配过程？
* String三种编码的使用规则是什么？
* `set`命令如何写入String和二进制数据的？
* `setbit`命令和`set`命令的实现过程有何区别？

阅读完本章后，你或许还会有上述等问题，那么请关注下一篇文章 ：《从实践中探究Redis原理》之String是字符数组吗（下）
