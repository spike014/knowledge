## 简介

`Redis` 是速度非常快的非关系型（`NoSQL`）内存键值数据库，可以存储键和五种不同类型的值之间的映射。

键的类型只能为字符串，值支持五种数据类型：字符串(`string`)、列表(`list`)、集合(`set`)、散列表(`hash`)、有序集合(`zset`)。

`Redis` 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。

## 数据类型

| 数据类型	| 可以存储的值	|操作|
| --------- | --- | ------- |
|STRING| 字符串，整数，浮点数	|对整个字符串或者字符串的其中一部分执行操作</br> 对整数和浮点数执行自增或者自减操作|
|LIST|	列表|	从两端压入或者弹出元素 </br> 对单个或者多个元素进行修剪，</br> 只保留一个范围内的元素|
|SET	|无序集合|添加、获取、移除单个元素</br> 检查一个元素是否存在于集合中</br> 计算交集、并集、差集</br> 从集合里面随机获取元素|
|HASH|	包含键值对的无需散列表|	添加、获取、移除单个键值对</br> 获取所有键值对</br> 检查某个键是否存在|
|ZSET|	有序集合|添加、获取、删除元素</br> 根据分值范围或者成员来获取元素</br> 计算一个键的排名|

### 字典 TODO

### 跳跃表 (有序集合)

是有序集合的底层实现之一。

跳跃表是基于多指针有序链表实现的，可以看成多个有序链表。

![](./image/beba612e-dc5b-4fc2-869d-0b23408ac90a.png)

在查找时，从上层指针开始查找，找到对应的区间之后再到下一层去查找。下图演示了查找 22 的过程。

![](./image/0ea37ee2-c224-4c79-b895-e131c6805c40.png)

与红黑树等平衡树相比，跳跃表具有以下优点：

- 插入速度非常快速，因为不需要进行旋转等操作来维护平衡性；
- 更容易实现；
- 支持无锁操作。

## 过期时间

Redis 可以为每个键设置过期时间，当键过期时，会自动删除该键。

对于**散列表**这种容器，只能为整个键设置过期时间（整个散列表），而**不能**为键里面的单个元素设置过期时间。

## 淘汰策略

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。

| 策略 | 描述 |
| :--: | :--: |
| noeviction | 到达最大的内存，新的键值对将不会被保存。当数据库使用复制时，这被用于主库|
| volatile-lru | 从已设置过期时间的数据集中挑选**最近最少使用**的数据淘汰 |
| volatile-ttl | 从已设置过期时间的数据集中挑选将要过期的数据淘汰 |
|volatile-random | 从已设置过期时间的数据集中任意选择数据淘汰 |
| allkeys-lru | 从所有数据集中挑选**最近最少使用**的数据淘汰 |
| allkeys-random | 从所有数据集中任意选择数据进行淘汰 |
| 以下是 4.0 新增 |  |
| volatile-lfu | 将**最少使用次数**的键与“到期”字段设置为 true。 |
| allkeys-lfu | 保留经常使用的钥匙；删除**最少使用次数**的（LFU）键 |

如果没有与先决条件相匹配的 `keys` (即已经设置过期时间的 `keys`)，则策略 volatile-lru, volatile-lfu, volatile-random, and volatile-ttl 的行为就会像 noeviction。

> https://redis.io/docs/manual/eviction/

使用 `Redis` 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 `allkeys-lru` 淘汰策略，将最近最少使用的数据淘汰。

这里要**注意** `LRU` 和 `LFU` 的区别：

> Let's consider a constant stream of cache requests with a cache capacity of 3, see below:
> 
> ```
> A, B, C, A, A, A, A, A, A, A, A, A, A, A, B, C, D
> ```
> 
> If we just consider a **Least Recently Used (LRU)** cache with a HashMap + doubly linked list implementation with O(1) > eviction time and O(1) load time, we would have the following elements cached while processing the caching requests as > mentioned above.
> 
> ```
> [A]
> [A, B]
> [A, B, C]
> [B, C, A] <- a stream of As keeps A at the head of the list.
> [C, A, B]
> [A, B, C]
> [B, C, D] <- here, we evict A, we can do better! 
> ```
> 
> When you look at this example, you can easily see that we can do better - given the higher expected chance of requesting > an A in the future, we should not evict it even if it was least recently used.
> 
> ```
> A - 12
> B - 2
> C - 2
> D - 1
> ```
> **Least Frequently Used (LFU)** cache takes advantage of this information by keeping track of how many times the cache > request has been used in its eviction algorithm.
> 
> https://stackoverflow.com/a/29225598/15042683

## 数据备份(持久化)与恢复

> https://redis.io/docs/manual/persistence/

### 方式

- `RDB`：以指定时间间隔对数据集进行快照
  - 缺点：
      - 如果系统发生故障，将会丢失最后一次创建快照之后的数据。

      - 如果数据量很大，保存快照的时间会很长。

- `AOF`：将写命令添加到 AOF 文件（`Append Only File`）的末尾。

    使用 AOF 持久化需要设置同步选项，从而确保写命令同步到磁盘文件上的时机。这是因为对文件进行写入并不会马上将内容同步到磁盘上，而是先存储到缓冲区，然后由操作系统决定什么时候同步到磁盘。有以下同步选项：

    | 选项 | 同步频率 |
    | :--: | :--: |
    | always | 每个写命令都同步 |
    | everysec(默认) | 每秒同步一次 |
    | no | 让操作系统来决定何时同步 |

    - `always` 选项会严重减低服务器的性能；
    - `everysec` 选项(默认)比较合适，可以保证系统崩溃时只会丢失一秒左右的数据；
    - `no` ：把你的数据交给操作系统。更快更不安全的方法。通常，Linux 会使用这种配置每30秒刷新一次数据，但这取决于内核的精确调优。

    建议(和**默认**)的策略是每秒 `fsync` 一次（`everysec`）。它既快速又相对安全。 `always` 策略在实践中非常慢，但是它支持组提交，所以如果有多个并行写操作，`Redis` 将尝试执行单个 `fsync` 操作。
    
    随着服务器写请求的增多，AOF 文件会越来越大。Redis 会自动执行 `AOF Rewrite`，重组AOF文件，降低其占用的存储空间。

### 方法

> https://redis.io/docs/manual/persistence/#backing-up-redis-data

- 备份

`RDB`
```shell
redis 127.0.0.1:6379> SAVE 
OK
```

`AOF`

在 `Redis` 配置文件中：

> mac: /usr/local/etc/redis.conf
> 
> linux(ubuntu):  /etc//redis/redis.conf
```
appendonly yes
```

- 恢复

如果需要恢复数据，只需将备份文件 (`dump.rdb`) 移动到 `redis` 安装目录并启动服务即可。获取 `redis` 目录可以使用 `CONFIG` 命令，如下所示：

```shell
redis 127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/usr/local/redis/bin"
```

- 官方建议实践：
  - 在服务器中创建 `cron` 作业，在一个目录中创建 RDB 文件的每小时快照，并在另一个目录中创建每日快照。
  - 每次运行 `cron` 脚本时，确保调用 `find` 命令来确保删除太旧的快照: 例如，您可以在最近的48小时内每小时拍摄快照，在一个月或两个月内每天拍摄快照。确保用日期和时间信息命名快照。
  - 每天至少确保传输一次 RDB 快照给你的数据中心外或者至少在运行 `Redis` 实例的物理机器之外的位置（个人注：用于外部保存备份）。

## 什么是缓存雪崩、缓存击穿、缓存穿透？TODO

**!!! 缓存：https://zhuanlan.zhihu.com/p/346651831**

**!!! 布隆过滤器 https://zhuanlan.zhihu.com/p/43263751**
