# Set一定是无序的吗-上

## 一、从一场线上事故说起

> 声明 ：故事纯属虚构，旨在从实际问题切入到文章主题，请不要过分讨论故事内容和技术细节。

### 1、初入江湖

小A计算机专业毕业后，入职了某互联网公司，主要负责某ToC产品的服务端开发。工作一段时间后，他迎来了第一个任务。

这天leader来到小A的工位上 ：”本周会起一版需求，由你负责，评审会快开始了，你去听下吧“ 。

”好的“ ，小A满怀期待的走进了会议室。

”本期需求很简单，就加一个在线相册功能“ 产品经理说后，开始介绍主要功能点 ：

* 用户可以上传照片到相册，并删除相册中的任一张照片
* 用户可以浏览相册中所有照片，按照时间倒序，最近上传的照片排在前面

”具体UI交互下面会串下流程，这个需求大家没问题吧“ 产品经理问道 。

小A发现需求确实挺简单就暗自开心的说道 ：”没问题“。

### 2、方案设计

需求评审结束后，小A又仔细完整过了下需求，列出了如下几个主要功能实现方案

(1) 照片的基本操作（上传、显示、删除）：直接使用公司内的基础图片服务

(2) 记录用户上传的照片 ：新建一个user\_album用户相册表

(3) 按顺序返回Photo Id列表 ：Redis缓存

”前两个都好实现，不过第三个中的Redis缓存使用什么结构更好呢“ 小A准备尝试下每个Redis类型。

当无心尝试Set类型的时候，小A惊奇的发现，`SMembers`返回的数据竟然是按照`SAdd`的顺序返回的。

![image](https://user-images.githubusercontent.com/35942268/167558237-e4c04b2a-868d-40cf-9c8f-aab4a2b2e360.png)

小A灵机一动，当每次用户上传照片时，执行一次`SAdd key photoId`，然后使用`SMembers`返回照片列表，再`reverse`返回就可以了。不但不用查库，而且还避免了可能会出现重复photoId的问题。伪代码如下

```go
// 上传照片
func uploadPhoto(userId int64, photoId string){
    redis.SAdd("key", photoId)
}

// getPhotoIdList 获取相册照片列表
func getPhotoIdList(userId int64) []string {
    return reverse(redis.SMembers(key))
}
```

### 3、开发上线

设计方案确定后，小A便开始投入开发，很快就通过测试，并顺利完成了上线。

### 4、线上事故

这天正在认真写代码的小A被一阵急促的声音打断了。

”线上有人反馈相册顺序乱了“，产品快走到了小A的身边。

”额，好我看下“，小A一边回答一边疑惑的分析问题原因：”不可能啊，就只执行了`SMembers`操作，当时测试也正常，为什么会乱序了呢？“

小A非常纳闷的开始查看线上数据，发现的确顺序全部乱了，还未等小A开始分析原因，leader就也找了过来。

”怎么这么多人都反馈相册顺序有问题了？“，leader问道。

”用户上传的照片Id都存在了Redis Set中，获取相册列表的时候直接用`SMembers`返回的，之前测试是正常的，不知道现在为什么会有问题了“ 小A尴尬的说道。

![image](https://user-images.githubusercontent.com/35942268/167635345-63df8511-2afa-410a-a656-3a4bcc9d09a1.png)

”你不知道Redis的Set底层结构吗？即使不知道，Set的定义就是无序的集合，为什么要做有序的事情呢，先抓紧时间修复下吧“，leader无奈的说道。

小A只能放弃了Set存储，把库中查出来的数据简单做了`jsonEncode`存到了String类型中，先紧急解决了线上问题，后续再优化逻辑 ……

### 5、提出问题

现在假如你是小A，如何找出这个问题的根本原因呢 ？

## 二、场景复现

### 1、实验

从上面的故事可知，Redis Set中的数据，应该是当成员数量达到某个值后，才会变成一个无序的结构。因此可以使用`eval "for i=1,N,1 do redis.call('sadd', 'key', i) end" 0` 命令，通过不断修改N的值，依次观察`SMembers`返回的数据顺序，找到让Set变成无序结构的临界值N。

| 测试次数   |        N | SMembers返回是否有序 |
| ------ | -------: | :------------: |
| 第1次测试  |   N = 10 |       有序       |
| 第2次测试  |  N = 100 |       有序       |
| 第3次测试  | N = 1000 |       无序       |
| 第4次测试  |  N = 500 |       有序       |
| 第5次测试  |  N = 700 |       无序       |
| 第6次测试  |  N = 600 |       无序       |
| 第7次测试  |  N = 550 |       无序       |
| 第8次测试  |  N = 510 |       有序       |
| 第9次测试  |  N = 512 |       有序       |
| 第10次测试 |  N = 513 |       无序       |

### 2、结论

通过上述实验发现，当 `N > 512` 时，Set结构就不再是一个有序的结构了。

![image](https://user-images.githubusercontent.com/35942268/167689561-3376176d-0203-4c8b-acb1-831a03a83fbf.png)

## 三、Set为什么要这样设计

### 1、问题在不断变化

在现实中，问题都是在不断变化的。所以当前问题下的解决方案，在下一时刻或许就不一定适用了。

### 2、选择最优的实现

如果熟悉数据库就会知道，有时候当数据量过小的时候，即使可以使用索引，数据库也会选择全表扫描，因为这种情况下使用全表扫描的性能，综合情况下是优于索引的。

所以对于Redis Set类型也是一样的，在成员数量过小的情况下，使用有序结构存储的性能，可能也是优于无序结构的。

## 四、从源码看Set底层原理

### 1、IntSet

在`src/intset.h`文件中定义了IntSet结构和操作API

![image](https://user-images.githubusercontent.com/35942268/167659357-5f3e9575-2c51-48a9-8083-abbd7381a8e7.png)

#### (1) 结构定义

* encoding ：编码类型，分为 `INTSET_ENC_INT16` / `INTSET_ENC_INT32` / `INTSET_ENC_INT64`
* length ：当前IntSet中contents的长度
* contents ：实际存储整型成员的数组

#### (2) 主要操作API

* `intset *intsetNew(void);` ：创建一个新的IntSet结构
* `intset *intsetAdd(intset *is, int64_t value, uint8_t *success);` ：如果不存在，则添加到contents中，否则Add不成功
* `intset *intsetRemove(intset *is, int64_t value, int *success);` ：基于二分查找的方式，删除value
* `uint8_t intsetFind(intset *is, int64_t value);` ：基于二分查找的方式，查找value

### 2、HashTable

在`src/dict.h`文件中定义了HashTable结构（又称为字典dictionary）和操作API

![image](https://user-images.githubusercontent.com/35942268/167677766-45a019e1-339a-4f1e-a29b-91093bd486f3.png)

#### (1) 结构定义

* type ：类型特定函数
* ht\_table\[2] ：存储数据的两个hash表，正常只有一个hash表中有数据，如果处于rehash状态，则两个hash表都有数据
* rehashidx ：rehash进度，如果未发生rehash，则为-1

#### (2) 主要操作API

* `dict *dictCreate(dictType *type)` ：创建一个新的HashTable
* `int dictAdd(dict *d, void *key, void *val)` ：添加一个新元素到HashTable中
* `int dictDelete(dict *d, const void *key)` ：搜索并删除一个元素

### 3、SAdd的过程

#### (1) SAdd命令

在`/src/commands.c`文件中的`redisCommandTable[]`数组中定义了`sadd`命令的详细内容。

![image](https://user-images.githubusercontent.com/35942268/167665669-39248267-e445-443a-8d40-ec0202ecdead.png)

其中就包括了`saddCommand()`函数，它定义在`/src/t_set.c`文件中，主要包括三个操作。

* 根据当前数据库和Key名，找到对应对象，并判断对象是否为Set类型
* 如果当前Set对象为NULL，则调用`setTypeCreate()`先创建一个Set对象，并关联到对应的数据库中
* 调用`setTypeAdd()`方法，使用`for循环`依次把SADD命令中携带的N个成员，加入到集合中

![image](https://user-images.githubusercontent.com/35942268/167666594-b850ba3d-cbb9-4011-873c-131f8c9ce502.png)

#### (2) 初始化创建

上面讲到如果Set对象为NULL，则`setTypeCreate()`会创建一个Set对象，让我们看下它的内部实现。可以发现，如果SADD的第一个成员为整型，那么会初始化一个IntSet结构的Set对象，否则会初始化一个HashTable结构的对象。

![image](https://user-images.githubusercontent.com/35942268/167672210-710a67d4-d8dd-4a13-b1c3-286fbb7987b3.png)

#### (3) 核心方法setTypeAdd

下面看`setTypeAdd()`这个最核心方法的实现，它定义在`/src/t_set.c`文件中。每当Add一个新成员时，都会先判断一下当前Set使用的编码类型。

* 如果是`OBJ_ENCODING_HT`编码，则会基于哈希表结构，完成Add操作
* 如果是`OBJ_ENCODING_INTSET`编码，则会基于IntSet结构，完成Add操作

![image](https://user-images.githubusercontent.com/35942268/167670846-d5d85b97-5df2-473e-a0d3-aff7324b0427.png)

#### (4) 从IntSet到HashTable的转换

需要特别注意的是，如果初始化Set时为IntSet结构，那么在每次Add时，都可能会触发从IntSet到HashTable的转换，由`setTypeConvert()`方法实现转换。以下为转换的条件 ：

* 当前Add的成员是否为整型（调用`isSdsRepresentableAsLongLong()`方法），一旦不是整型，就会转换为HashTable
* 如果Inset中Contents长度（调用`intsetLen()`方法）超过了设置的 `server.set_max_intset_entries`（默认512），就会转换为HashTable

## 五、未完待续

* IntSet的顺序是升序、降序、还是按照插入的顺序？
* IntSet结构的详细工作原理是什么？
* IntSet结构相比于HashTable的优势是吗？

阅读完本章后，你或许还会有上述等问题，那么请关注下一篇文章 ：《从实践中探究Redis原理》之Set一定是无序的吗（下）
