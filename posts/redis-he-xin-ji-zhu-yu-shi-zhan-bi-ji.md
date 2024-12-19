---
title: 'Redis核心技术与实战笔记'
date: 2024-12-07 20:23:23
tags: [redis,geektime,note]
published: true
hideInList: false
feature: /post-images/redis-he-xin-ji-zhu-yu-shi-zhan-bi-ji.png
isTop: false
---
## 数据类型和数据结构

| 数据类型   | 数据结构                             |
| ---------- | ------------------------------------ |
| String     | SDS（简单动态字符串）                |
| List       | 双向链表、压缩列表(ZipList)          |
| Hash       | 压缩列表(ZipList)、哈希表(HashTable) |
| Set        | 整数数组(IntSet)、哈希表(HashTable)  |
| Sorted Set | 压缩列表(ZipList)、跳表(SkipList)    |

### 压缩列表

压缩列表类似数组，不同的是压缩列表表头有3个字段`zlbytes`、`zltail`、`zllen`，分别表示列表长度（字节数）、列表尾元素偏移量、列表中元素个数。此外列表尾还有一个`zlend`表示列表结束，固定值`0xFF`。
<!-- more -->

![](https://xuhang.github.io/post-images/1733574319043.png)

压缩列表只有查找第一个元素、最后一个元素获取列表长度的时间复杂度为`O(1)`，查找其他元素为`O(N)`。

```
area        |<------------------- entry -------------------->|

            +------------------+----------+--------+---------+
component   | pre_entry_length | encoding | length | content |
            +------------------+----------+--------+---------+
```

pre_entry_length:

- `1` 字节：如果前一节点的长度小于 `254` 字节，便使用一个字节保存它的值。
- `5` 字节：如果前一节点的长度大于等于 `254` 字节，那么将第 `1` 个字节的值设为 `254` ，然后用接下来的 `4` 个字节保存实际长度。

Redis 基于压缩列表实现了 List、Hash 和 Sorted Set 集合类型，可以节省内存空间。

### 跳表

本质是一个有序链表，通过增加多级索引，实现数据的快速定位，时间复杂度为`O(LogN)`。

### Hash

Redis Hash 类型有两种底层实现，分别是压缩列表和哈希表。

`hash-max-ziplist-entries`：用压缩列表保存哈希时的最大元素个数，默认 512

`hash-max-ziplist-value`：用压缩列表保存哈希时单个元素的最大长度，默认 64

二者满足一个超过阈值，Redis 就会自动把 Hash 的实现由压缩列表转为哈希表，之后就会一直用哈希表进行保存，不会转回为压缩列表。

当使用 `REDIS_ENCODING_ZIPLIST` 编码哈希表时， 程序通过将键和值一同推入压缩列表， 从而形成保存哈希表所需的键-值对结构。

![](https://xuhang.github.io/post-images/1733574355472.png)

新添加的 key-value 对会被添加到压缩列表的表尾。

当进行查找/删除或更新操作时，程序先定位到键的位置，然后再通过对键的位置来定位值的位置。

## Redis 6.0新特性

### 多线程处理网络请求

只是用多线程处理网络请求，读写命令仍然使用单线程。可以有效解决网络IO的瓶颈，提升整体性能。

```ini
io-threads-do-reads yes
# 启用多线程处理IO

io-threads 6
# 线程数
```

### Tracking

业务应用中的Redis客户端可以把读取的数据缓存在业务应用本地。

**普通模式**：实例会在服务端记录客户端读取过的key，并监控key是否有修改，如果发送变化，服务端会给客户端发送`invalidate`消息，通知客户端缓存失效。但是该消息只会发送一次，只有客户端再次从服务端读取数据时，可能再次发送。通过命令`CLIENT TRACKING ON|OFF`启用或关闭。

**广播模式**：服务端会给客户端广播所有key的失效情况，如果key被频繁修改，可能会消耗大量网络带宽。

```shell
CLIENT TRACKING ON BCAST PREFIX user
# 客户端指定监听的key的前缀，这个前缀的key被修改后，服务端会给客户端发送invalidate消息。需要key的命名规范。
```

上面的两种模式需要使用6.0新增的`RESP 3`通信协议，对于使用`RESP 2`	协议的客户端，需要使用重定向模式，即客户端需要使用`SUBSCRIBE`命令订阅`_redis_:invalidate`频道用于监听key失效的消息，同时还需要执行`CLIENT TRACKING`命令，用于接收`RESP 2`客户端重定向过来的消息。

### 权限控制

6.0之前只能通过设置密码来控制安全访问，6.0提供了更加细粒度的访问权限控制。

**用户**：6.0之前没有用户概念，6.0可以创建用户并单独设置密码

```shell
ACL SETUSER <USER-NAME> on > <PSSWORD>
```

**权限**：可以将具体的操作权限，赋予给用户或撤销

```shell
ACL SETUSER <USER-NAME> +@hash -@string
```

|      命令      | 说明                                   |
| :------------: | -------------------------------------- |
|  `+<COMMAND>`  | 将==一个命令==添加到用户的可执行列表中 |
|  `-<COMMOND>`  | 将==一个命令==从用户可执行列表中移除   |
| `+@<CATEGORY>` | 将==一类命令==添加到用户的可执行列表中 |
| `-@<CATEGORY>` | 将==一类命令==从用户可执行列表中移除   |
|    `+@ALL`     | 运行执行==所有==命令                   |
|    `-@ALL`     | 禁止执行==所有==命令                   |

还可以以key的粒度设置访问权限。

```shell
ACL SETUSER <USER-NAME> ~KEY:* +@ALL
```

允许用户对KEY为前缀的key的所有操作。

## rehash

Redis使用2个全局哈希表，类似JVM堆内存中的Survivor区的S1和S2，始终只使用其中一个，rehash过程中使用另一个进行替换。

1. 给哈希表2分配更大的空间
2. 把哈希表1中的数据重新映射并拷贝到哈希表2中
3. 释放哈希表1的空间

但是步骤 2 在拷贝数据时，如果一次性拷贝大量数据会造成Redis线程阻塞，无法服务其他请求。为避免这个问题，Redis采用了**<u>渐进式rehash</u>**。

具体步骤是在第2步拷贝数据时，仍然正常处理客户端请求，同时从原哈希表中的第一个索引位置开始，连同本次请求所在的索引位置上的所有entries拷贝到新的哈希表中，对于新增操作只会在新的哈希表中新增，保证原哈希表逐渐减小。但是即使没有新的请求，Redis也会以一定的频率定时地执行一次rehash，且每次时长不超过1ms。

rehash时机：

load factor >= 1 且 没有正在进行RDB生成和重新AOF

load facotr >= 5 不管有没有正在进行RDB生成或重新AOF

## bigkey

通过命令`./redis-cli --bigkeys -i 0.1 -a 密码`来查看整个数据库中bigkey情况，该命令会输出每种类型的键值对个数和平均大小，还会输出每种数据类型中最大的bigkey信息。因为这个工具会扫描整个数据库，可能会对Redis性能产生影响，所以最后是在从库上执行，同时加上`-i`选项指定扫描间隔，避免长时间的扫描。但是整个命令只返回了每个数据类型中最大的bigkey，对于集合类型是元素个数，无法查看Top-N的情况。所以最好是通过`SCAN`命令扫描数据库，然后用`TYPE`命令获取值的类型；对于集合类型，使用`LLEN(List)`、`HLEN(Hash)`、`SCARD(Set)`、`ZCARD(Sorted Set)`获取元素个数，对于字符串，使用`STRLEN`获取字符串长度；或者使用`MEMROY USAGE <KEY>`获取占用的内存空间。

## 高性能的IO模型

通常说的Redis是单线程的，主要是值Redis的网络IO和键值对的读写是由一个线程完成的。Redis在6.0中加入了多线程IO，提高网络请求处理的并行度，但是仅仅是对于网络请求的处理是使用多线程，命令的读写仍然是单线程。

## AOF日志

AOF日志是写后日志，即先执行命令，数据写入内存后再记录日志。AOF记录的是Redis收到的每一条命令，以文本形式保存。

比如命令`set testkey testvalue`，其AOF日志形式为：

| AOF记录   | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| *3        | 当前命令有三部分组成，每部分由`$数字`开头，其中数字表示命令、键、值的字节长度 |
| $3        | 这部分包含3字节命令，也就是set                               |
| set       | 命令                                                         |
| $7        | 7字节，也就是testkey                                         |
| testkey   |                                                              |
| $9        | 9字节，也就是testvalue                                       |
| testvalue |                                                              |

但是为了避免额外的检查开销，Redis不会检查这些命令的语法，所以只有先执行命令，命令执行成功后才会记录AOF日志，这样可以保证记录的是正确的命令。此外还有一个好处是不会阻塞当前的写操作（可能会阻塞下一个操作，因为AOF日志是在主线程中执行的）。

AOF的写回策略：

| 策略         | 过程                                                         | 说明                               |
| ------------ | ------------------------------------------------------------ | ---------------------------------- |
| **Always**   | 同步写回，每个写命令执行完，立即同步写回磁盘                 | 基本不丢数据，但是会影响主线程性能 |
| **Everysec** | 每秒写回，每个写命令执行完，先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区内容写入磁盘 | 宕机丢失数据                       |
| **No**       | 由操作系统控制写回，每个写命令执行完，只把日志写到AOF文件缓冲区，由操作系统决定何时将内容写回磁盘 | 在性能和数据丢失取折中             |

AOF文件过大导致的性能问题：

1. 文件系统对文件大小有限制，不能无限大
2. 过大的文件写入性能会受到影响
3. 发生宕机利用AOF记录恢复数据，文件过大会导致整个过程很慢

### AOF重写

解决AOF文件过大的问题，重写时，根据Redis数据库现状生成一个新的AOF文件。对一个键进行的多次写操作，最终会生成一条写命令，可以有效减小AOF文件的大小。AOF重写过程由后台线程`bgrewriteaof`完成，可以避免阻塞主线程。

**一个拷贝，两处日志**：主线程fork出后台的`bgrewriteaof`子进程，会把主线程的内存**拷贝（父进程的页表，而非物理内存）**一份给`bgrewriteaof`子进程，子进程把拷贝的数据计入重写日志。同时，新的写操作会写入**原来的AOF日志**（完整的AOF）和**新的AOF重写日志**（等拷贝生成的重写日志完成后，追加上这个重写期间的增量日志，凑成一份完整的日志，替换原来的AOF日志）

## 内存快照RDB

使用AOF进行故障恢复需要把操作日志重新执行一遍，耗时较长，使用内存快照将大大提升恢复速度。内存快照指的是内存中的数据在某一时刻的状态。

Redis提供了两个命令生成RDB文件：`save` （在主线程中执行，会阻塞）和 `bgsave`（fork子进程执行，默认）

`bgsave`执行过程中，主线程仍能处理写请求，Redis会借助操作系统提供的写时复制(Copy-on-write, COW)正常处理写请求。

如果频繁地生成RDB文件，会给磁盘带来很大压力，而且fork创建`bgsave`子进程的创建过程会阻塞主线程，影响Redis性能。

Redis4.0提出了AOF和内存快照的混合使用：内存快照以一定的频率执行，两次快照期间使用AOF日志记录。

## 主从模式

通常说Redis具有高可靠性，是指数据尽量少丢失、服务尽量少中断。前者通过AOF和RDB保证，后者则是通过冗余副本，即一份数据保存在多个实例上。

主从读写分离中，写操作只在主库上执行，然后同步到从库，读操作则是主从库都可以。

通过命令设置从库：

```shell
replicaof MASTER_IP MASTER_PORT
# 5.0之前使用slaveof命令
```

1. 从库向主库发送命令`psync ? -1`进行全量同步，命令的第一个参数是主库的`runID`，由于第一次复制从库不知道主库的`runID`，用`?`表示，第二个参数`-1`表示第一次复制；
2. 主库响应`FULLRESYNC {runID} {offset}`给从库，`runID`为主库的实例ID，offset表示复制进度
3. 主库执行`bgsave`生成RDB文件，并发送给从库，同时生成RDB文件过程中的写操作记录到`replication buffer`中
4. 从库收到RDB文件后先清空当前数据库，然后加载RDB文件
5. 主库将`replication buffer`发送给从库，从库执行操作

如果从库数量很多，且都要和主库进行全量复制，主库就会忙于fork子进程生成RDB文件，且占用主库的网络带宽，可以采用“主-从-从”的模式，以级联的模式将压力分散到从库上。

```shell
replicaof SLAVE_IP SLAVE_PORT
```

一旦完成主从全量复制之后，它们之间就会一直维护一个网络长连接，主库通过这个连接将后续收到的命令同步给从库，长连接可以避免频繁建立连接的开销。

如果发送网络断连，Redis2.8之后，主从库会采用增量复制的方式继续同步。主库会把断连期间收到的写命令，写入`replication buffer`和`repl_backlog_buffer`缓冲区，`repl_backlog_buffer`是一个环形缓冲区，主库记录自己写到的位置(`master_repl_offset`)，从库记录自己读到的位置(`slave_repl_offset`)。主从库的连接恢复之后，从库给主库发送`psync`命令，带着`slave_repl_offset`，主库则将`slave_repl_offset`~ `master_repl_offset`之间的命令操作同步给从库。

> 参数`repl_backlog_size`表示环形缓冲区的大小

### 主从同步的问题

主从数据不一致是指客户端从从库读到的值和主库中的最新值不一致。因为主从库间的命令是异步进行的，另外，主从库间的网络可能有延迟，导致从库接收到主库的同步命令延后；或者从库正在进行复杂度高的命令被阻塞，需要等命令结束了才能继续执行主库同步过来的命令。

`INFO replication`命令可以查看主库接收写命令的进度信息(`master_repl_offset`)和从库同步写命令的进度信息(`slave_repl_offset`)，我们可以通过这个命令监控主从的同步进度，对于同步进度差太多的从库可以选择从客户端的连接信息中移除，等恢复正常了再添加进来。

客户端可能在从库上读到过期的数据，Redis 采用惰性删除和定期删除的策略删除过期的数据，客户端在访问过期数据时，如果数据已过期，则会被删除，给客户端返回空；同时，Redis 会定期随机选出一定数量的数据，检查是否过期，并把过期的数据删除。但是客户端从从库读取到过期数据时，是不会主动删除的。（这个问题在 Redis3.2 之后的版本优化，从库读到过期数据，虽然不会删，但是返回的是空值）另外，如果设置过期时间使用的是`EXPIRE key 60`，命令同步到从库，再到执行，可能会延后一段时间，导致从库上的有效期在主库之后；如果使用的是`EXPIREAT key <TIMESTAMP>`，也可能由于主从节点上的时钟不一致，导致数据的过期时间不一致。不过，还是尽量使用`EXPIREAT`命令设置过期时间，同时主从节点和相同的 NTP 进行时钟同步。

> 将`slave-serve-stale-data`设置为 no，这样从库就只能服务 INFO、SLAVEOF 命令，避免从库执行写命令导致数据不一致。
>
> `slave-read-only`为 yes 时，从库只能处理读请求，无法处理写请求，二者有区别

### 脑裂

在主从集群中，同时有两个主节点，都能接收写请求。结果是客户端可能往不同的主节点上写数据，导致数据丢失。

如果是主库的数据还没同步到从库，主库故障，发生主从切换导致的数据丢失， 可以通过原主库的`master_repl_offset`和原从库的`slave_repl_offset`的差值来判断。——如果offset一致，则说明丢数据不是由脑裂导致的。

脑裂发生可能是由于主库“假故障”导致的，主库CPU满载，导致无法响应心跳，哨兵会将主库标记为客观下线，然后开始执行主从切换。但是主库在切换过程中恢复正常，可以正常处理请求，主从切换完成后，客户端的写命令才会发送到新的主库上，主库也开始执行`slave of`和新主库进行全量同步，清空本地数据，导致数据丢失。

通过参数`min-slaves-to-write`和`min-slaves-max-lag`限制至少有N个从库和主库进行数据复制时的ACK消息延迟不超过T秒，主库才会正常接收客户端请求。如果主库“假故障”时，无法响应哨兵心跳，也就无法保证上面的限制，主库就不再接收客户端请求了。

## 哨兵机制

哨兵机制是实现主从库自动切换的关键机制，有效解决主从复制模式下故障转移的问题。

哨兵是运行在特殊模式下的Redis进程，通常以集群的模式运行，哨兵主要负责三个任务：监控、选主、通知。

```shell
#配置哨兵
sentinel monitor MASTER_NAME MASTER_IP MASTER_PORT QUORUM
```

主库上有一个名为`__sentinel__:hello`的频道，所有的哨兵都会订阅这个频道，实现相互通信。哨兵会将自己的IP和端口发布到`__sentinel__:hello`频道，其他哨兵就只到了新哨兵的地址和端口，然后和其建立连接。

哨兵和客户端进行信息同步通用是通过pub/sub机制

|                 事件                  |          频道          |
| :-----------------------------------: | :--------------------: |
|           主库下线-主观下线           |        `+sdown`        |
|         主库下线-退出主观下线         |        `-sdown`        |
|           主库下线-客观下线           |        `+odown`        |
|         主库下线-退出客观下线         |        `-odown`        |
|   从库重新配置-哨兵发送SLAVEOF命令    |  `+slave-reconf-sent`  |
| 从库重新配置-从库配置新主库，尚未完成 | `+slave-reconf-inprog` |
|   从库重新配置-从库配置新主库，完成   |  `+slave_reconf-down`  |
|               主从切换                |    `+switch-master`    |

哨兵把新的主库选出来后，客户端根据订阅得到的事件(`switch-master`)和新主库进行通信

```shell
switch-master MASTER_NAME OLD_IP OLD_PORT NEW_IP NEW_PORT
```

哨兵会向主库发送`INFO`命令，该命令会返回从库列表，哨兵根据返回的信息和从库建立连接，通过此连接持续地对从库进行监控。

哨兵在运行时，周期性地给所有**主从**库发送PING命令，检查它们是否在线，如果没有在规定时间内响应哨兵的PING命令，哨兵就会将其标记为“主观下线”，如果是主库“主观下线”，该哨兵就会给其他哨兵实例发送`is-master-down-by-addr`命令，其他实例会根据自己和主库的连接情况，响应`Y` 或 `N`，如果得到的赞成票达到配置的`QUORUM`值，就可以将主库标记为“客观下线”。然后哨兵给其他哨兵发送命令，表明希望由自己来执行主从切换，其他哨兵投票，这个过程称为“Leader选举”。但是同一时间可能有多个哨兵判断主库为“客观下线”，多个哨兵都会发起“Leader选举”，其他哨兵会给第一个向其发送命令的哨兵投Y，给后续其他哨兵投N。最终拿到赞成票`>= N/2 + 1 && >= QUORUM`的哨兵当选为“Leander”来执行主从切换。如果不满足“Leader”票数，哨兵集群会等待一段时间(哨兵故障转移时间*2)，再次重新选举。

> 保证所有哨兵实例的配置是一致的，否则可能由于主观下线的时间(down-after-milliseconds)不一致导致主库故障切换不及时。

## 切片集群

如果Redis数据库数据量特别大，占用很大内存，会导致RDB非常耗时，同时fork创建子进程阻塞主线程的时间也会相对较长。可以使用切片集群，或者分片集群，即启动多个Redis实例组成一个集群，按照一定的规则将数据划分成多份，每一份保存在一个实例上。这样每一个实例生成RDB的文件就小了很多。

纵向扩展(scale up)：升级单个Redis实例的配置，如内存、磁盘，CPU

横向扩展(scale out)：增加Redis实例个数

切片集群是一种保存大量数据的通用机制，Redis3.0之后，官方提供了名为Redis Cluster的实现方案。Redis Cluster方案采用哈希槽(Hash Slot)来处理数据和实例之间的映射关系。一个切片集群共有16384($2^{14}$)个哈希槽，类似于数据分区，每个数据都会根据其key(CRC16算法)被映射到一个槽中。

Redis实例会把自己的哈希槽信息发送给和它相连的其他实例，来完成哈希槽分配信息的扩散，实例之间互联后，所有实例就都有哈希槽的映射关系了。客户端收到哈希槽信息后会缓存在本地，请求时先计算键对应的哈希槽，然后给响应的实例发送请求。但是如果集群实例有删减、或者实例负载不均衡都可能需要重新分配哈希槽。重新分配之后，各实例之间仍然可以通过连接相互传递分配信息实现同步，但是客户端无法感知这些变化。所以Redis Cluster提供了一种重定向机制，即客户端按原来的流程给实例发送命令，但是实例上没有该键映射的哈希槽，就会返回一个`MOVED`响应，包含新实例的地址和端口，客户端重新向新实例发送请求，同时更新缓存的映射关系。

使用`cluster create`命令创建集群，Redis默认会把这些槽平均分布在集群实例上。也可以通过命令`cluster meet`手动建立实例间的连接，再用`cluster addslots`指定每个实例上的slot。

```shell
#加入集群
redis-cli -p 6379 cluster meet 127.0.0.1 6380
# 查看集群节点信息
cluster nodes
cluster info
# 分配槽
redis-cli -h IP -p PORT cluster addslots {0..5461}
```

**重新分片的流程**

1. 对目标节点发送`cluster setslot <slot> importing <source-node-id>`命令，让目标节点准备导入槽的数据。

2. 对源节点发送`cluster setslot <slot> migrating <destination-node-id>`命令，让源节点准备迁出槽的数据。

3. 源节点循环执行`cluster getkeysinslot {slot} {count}`命令，获取count个属于槽{slot}的键。

4. 对于步骤3中获取的每个key，`redis-trib.rb`都向源节点发送一个`MIGRATE <target_ip> <target_port> <key_name> 0 <timeout> `命令，将被选中的键原子性地从源节点迁移至目标节点。

5. 重复执行步骤3和4，直到源节点保存的所有属于槽slot的键值对都被迁移到目标节点为止。

6. `redis-trib.rb`向集群中的任意一个节点发送`CLUSTER SETSLOT <slot> NODE <node-id>`命令，将槽slot指派给目标节点。这一消息会发送给整个集群。

**客户端ASK重定向流程**

Redis集群支持在线迁移slot和数据来完成水平伸缩，当slot对应的数据从源节点到目标节点迁移过程中，客户端需要做到智能识别，保证键命令可正常执行。例如，当一个slot数据从源节点迁移到目标节点时，可能会出现一部分数据在源节点，另一部分在目标节点。

如果出现这种情况，客户端键执行流程将发生变化，如下所示，

1. 客户端根据slot缓存发送命令到源节点，如果存在key则直接执行并返回结果。

2. 如果key不存在，则可能存在于目标节点，这时会回复ASK重定向异常，格式如下：`(error) ASK {slot} {targetIP}:{targetPort}`。

3. 客户端从ASK重定向异常提出目标节点信息，发送asking命令到目标节点打开客户端连接标识，再执行键命令。如果存在则执行，不存在则返回不存在信息。

ASK与MOVED虽然都是对客户端进的重定向，但是有着本质区别，前者说明集群正在进行slot数据迁移，所以只是临时性的重定向，不会更新slot缓存，但是MOVED重定向说明键对应的槽已经明确指定到新的节点，会更新slot缓存。

## 字符串

简单动态字符串 SDS 的结构：

```c
struct __attribute__((__packed)) hisdshdr8 {
    uint8_t len;
    uint8_t alloc;
    unsigned char flags;
    char buf[];
}
```

Redis 中字符串有 5 种结构体：~~`hisdshdr5`~~、`hisdshdr8`、`hisdshdr16`、`hisdshdr32`、`hisdshdr64`，分别表示 len 和 alloc 占用的位数，其中`len`表示 buf 已用长度，`alloc`表示分配的长度，`flags`表示字符串类型，`buf`数组存储实际的数据，为了表示数组的结束，Redis 会自动在数组最后添加`'\0'`，会额外占用一个字节。对于 String 类型来说，除了 SDS 的开销，还有一个来自`RedisObject`结构体的开销。

```c
typedef struct redisObject {
    unsigned type:4; // 数据类型，如字符串、列表、哈希等
    unsigned encoding:4; // 编码方式，如 int、raw、hashtable 等
    unsigned lru:LRU_BITS; // Least Recently Used，用于记录对象最近被访问的时间
    int refcount; // 引用计数，用于自动内存管理
    void *ptr; // 指向实际存储数据的指针
} robj;
```

当保存的是 Long 类型整数时，RedisObject 中的指针就直接赋值为整数数据，就不用额外指向整数了（int 编码）；当保存的是字符串，并且长度小于等于 44 字节时，RedisObject 中元数据、指针和 SDS是一块连续的内存区域（embstr 编码）；当保存的字符串长度大于44 字节时，RedisObject 和 SDS 就不在连续的内存区域了，而是给 SDS 分配独立的空间，并用指针指向它（raw 编码）

![image-20241009210128383](%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90%E6%96%87%E4%BB%B6%E5%A4%B9/image-20241009210128383.png)

> 字符串长度小于等于 44 字节时，使用 hisdshdr8即可保存，根据hisdshdr8的结构体可知，SDS 占用内存：1(len) + 1(alloc) + 1(flags) + 44(字符串) + 1('\0') = 48 Bytes，RedisObject 占用内存：8(元数据) + 8 (指针) = 16Bytes，一共 48+16 = 64 字节。

Redis 使用的内存分配库是`jemalloc`，分配内存时，会根据申请的字节数N，找一个>=N，但是最接近 N 的 $2^{n}$数作为分配的空间。

## 消息队列

消息队列的3个基本需求：消息保序、重复消息处理、消息可靠性。

保序就是消费者需要按照生产者发送到消息队列的顺序处理消息；

重复消息处理是在网络堵塞时出现消息重传的情况，消费者可能会收到多条重复的消息；

消息可靠性是消费者出现消息时可能出现故障导致宕机，消息没有处理完成的情况；

Redis的List和Stream两种数据类型满足作为消息队列的三种需求。

List作为队列使用时是按照先进先出的顺序进行存取的，比如`LPUSH`写入消息，`RPOP`消费消息，如果消息队列为空，则客户端需要循环地去请求。Redis提供了`BRPOP`命令，如果消息队列为空，客户端则会阻塞直到有消息写入队列。对于消息的重复处理，则是靠生产者和消费者约定的全局唯一ID实现，生产者给每个写入的消息一个全局唯一ID，消费消费时则记录处理过的ID，对每个ID的消息只处理一次。Redis对消息的可靠性是通过命令`BRPOPLPUSH`命令实现的，这个命令让消费者从队列中读取消息的同时，把这个消息插入到另一个List中备份。当宕机重启后，没处理完成的消息从备份列表中读取再次进行处理。`BRPOPLPUSH MQ_LIST BACKUP_LIST TIMEOUT`

> Redis在6.2.0版本只会引入了BLMOVE用来替代上面的命令

Streams是Redis5.0专为消息队列设计的数据类型。

```shell
XADD MQ_LIST * KEY VALUE
# 向消息队列中加入消息，*表示由Redis生成全局唯一ID，也可以自行设定，但是要保证全局唯一
XREAD BLOCK 100 STREAMS MQ_LIST <ID>|$
# 从指定的ID开始，读取后面的所有消息，$表示读取最新消息，BLOCK表示阻塞读，阻塞超时时长100ms
```

Streams支持以消费组的形式消费消息

```shell
XGROUP create MQ_LIST <GROUP_NAME> 0
# 创建消费组消费消息队列MQ_LIST
XREADGROUP group <GROUP_NAME> <CONSUMER_NAME> streams MQ_LIST >
# 让组内的消费组从MQ_LIST中读取消息，> 表示从第一条未被消费的消息开始读取
```

为了保证消费组在故障或宕机重启后，仍然能读取未处理完的消息，Streams会自动使用内部队列(PENDING list)留存组里每个消费者读取的消息，直到消费者用`XACK`命令通知Streams消息已处理完。消费者重启后，可以用`XPENDING`命令查看已读取但未ACK的消息。

```shell
XPENDING MQ_LIST <GROUP_NAME> - + 10 <CUSOMER_NAME>
# 查询某个消费者已读取，但是未ACK的消息
XACK MQ_LIST <GROUP_NAME> <ID>
```

## 异步机制

| 阻塞点                                    | 阻塞结果                                            | 能否异步处理                                                 |
| ----------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| 网络IO                                    | 使用IO多路复用，不会阻塞主线程                      |                                                              |
| 增删改查                                  | ✖️O(N)复杂度的操作，删除大集合（bigkey）会导致阻塞   | ✖️读操作不能异步，但是删除只用返回给客户端OK，可以异步进行（惰性删除） |
| 数据库操作                                | ✖️清空数据库（FLUSHDB、FLUSHALL）会阻塞              | 只用返回给客户端OK，可以异步进行                             |
| 生成RDB文件                               | fork子进程进行，创建子进程过程会短暂阻塞，影响不大  |                                                              |
| 记录AOF文件                               | ✖️always同步写回策略会阻塞主线程                     | 写回策略改为everysec，Redis会启动子线程来执行AOF日志的写盘   |
| 重写AOF                                   | fork子进程进行，创建子进程过程会短暂阻塞，影响不大  |                                                              |
| 主从复制-生成、传输RDB文件                | 子进程处理，不阻塞主线程                            |                                                              |
| 主从复制-从库接收RDB、清空数据库、加载RDB | ✖️FLUSHDB清空数据库会阻塞，加载RDB会阻塞             | ✖️加载RDB必须主线程执行，不能异步                             |
| 切片集群-同步哈希槽信息                   | 哈希槽信息量不大，影响不大                          |                                                              |
| 切片集群-数据迁移                         | Redis Cluster方案使用同步迁移，在迁移bigkey时会阻塞 |                                                              |

> 异步的键值对删除和数据库清空是Redis4.0之后提供的.
>
> 删除大量元素的集合时，使用`UNLINK`，清空数据库使用`FLUSHDB ASYNC`和`FLUSHALL ASYNC`

## Redis变慢的原因

### 慢查询命令

在Redis中执行速度慢的命令，会导致Redis延迟增加。

针对慢查询的命令，可以选择其他高效的命令进行替代，比如`SSCAN`多次迭代替换`SMEMBERS`用于查询Set中所有成员。或者针对Set集合的排序、交集、并集可以在客户端完成。

> `KEYS *`命令会遍历所有键值对，延时高，应该避免使用

```ini
slowlog-log-slower-than
# 慢查询日志对执行时间大于多少微秒的命令进行记录

slowlog-max-len 1000
# 慢查询日志最多能记录1000条命令，默认128
```

使用`SLOWLOG GET`命令查看慢查询日志中记录的操作命令。Redis2.8.13开始提供`latency monitor`监控工具，用来监控Redis运行期间峰值延迟情况，首先设置要监控的阈值`config set latency-monitor-threshold 1000`，然后使用命令`latency latest`命令查看超过1000微秒的延迟情况。

### 过期Key操作

默认情况下，Redis每100ms会删除一些过期的key。步骤如下：采样`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOK`个数（默认20）的key，将其中过期的key删除；如果过期的key占比找过`25%`，则重复此步骤，直到占比低于`25%`。

> 除了上面的定时任务，当访问一个键时，也会对键的过期时间进行检查，如果过期，就将键删掉。
>
> 两种方式删除过期键时，都会产生一个expired通知，产生 expired 通知的时间为过期键被删除的时候， 而不是键的生存时间变为 0 的时候。

如果使用`EXPIREAT`命令设置大量同一时间过期的键，会导致Redis一直忙于删除过期键来释放空间，这个过程会阻塞主线程，导致时延增加。所以在使用`EXPIREAT`或`EXPIRE`设置过期参数时加一个可以接收的随机数。

### AOF写盘

AOF重写时，会进行大量的磁盘IO，可能会导致AOF日志(`everysec`或`always`策略)的`fsync`被阻塞，虽然`fsync`由子线程执行，但是主线程会监控`fsync`的进度，如果上一次的`fsync`未完成，主线程就会阻塞。

`no-appendfsync-on-rewrite yes`配置（默认`no`）表示在AOF重写时不进行`fsync`操作，但是此时宕机会导致数据丢失。

### swap

内存swap是操作系统将内存数据在内存和磁盘间来回换入换出的机制，涉及磁盘的读写，影响性能。当Redis实例占用大量内存，或者同一机器上运行其他进程占用大量内存时，会导致分配给Redis实例的内存不足，进而触发swap。

## 内存碎片

`INFO memory`命令用于查看内存使用情况，`used_memory`是Redis为保存数据实际使用的空间，`used_memory_rss`是操作系统实际分配给Redis的空间，`mem_fragmentation_ratio = used_memory_rss / used_memory` 是表示当前内存碎片率的指标。

Redis从4.0-RC3版本开始，提供了自动清理内存碎片机制，通过命令`config set activedefrag yes`开启该机制。同时满足下面两个配置参数时自动触发内存清理：

```ini
# 内存碎片字节数达到100MB
active-defrag-ignore-bytes 100mb
# 内存碎片空间占比达到 10%
active-defrag-threshold-lower 10
```

但是内存碎片清理时，需要把多份数据拷贝到新的位置，会阻塞其他操作。

```ini
# 内存清理过程中所用CPU时间比例最低 25%， 保证清理能正常开展
active-defrag-cycle-min 25
# 内存清理过程中所用CPU时间占比最高 75%， 保证不影响其他操作
active-defrag-cycle-max 75
```

## 缓冲区

服务器给每个连接的客户端配置了一个输入输出缓冲区，输入缓冲区会把客户端发过来的命令暂存起来，Redis主线程从输入缓冲区读取命令再执行；执行完成后将结果写入输出缓冲区，返回给客户端。

### 输入缓冲区

```shell
client list

# 查看和Redis服务器相连的每个客户端对输入缓冲区的使用情况
# ... qbuf=xxx qbuf-free=xxx cmd=xxx
# qbuf表示缓冲区已使用大小，qbuf-free表示缓冲区未使用的大小，cmd表示最新执行的命令
```

当客户端写入`bigkey`，或者Redis主线程阻塞无法及时处理客户端命令，导致堆积在缓冲区溢出，Redis就会将连接关闭。输入缓冲区的大小无法调整，Redis设定的是`1G`。

### 输出缓冲区

Redis为每个客户端设置的输出缓冲区包含两部分：大小固定为`16KB`的固定缓冲区，暂存`OK`和出错信息；可以动态增加的缓冲区，存放大小可变的响应结果。

`MONITOR`命令是用来持续监测Redis执行的，该命令的输出会持续占用输出缓冲区，最终发生溢出，所以不要在生产中使用。

输出缓冲区可以通过参数`client-output-buffer-limit`来进行配置

```ini
# normal: 普通client,包括monitor 的 buffer限制 ，3个0表示不做限制
client-output-buffer-limit normal 0 0 0
# slave：主从的slave client buffer限制，超过256m，或者超过64m持续60s，则关闭客户端连接
client-output-buffer-limit slave 256mb 64mb 60
# pubsub：pubsub模式中的 client buffer限制，超过32m，或者超过8m持续60s，则关闭客户端连接
client-output-buffer-limit pubsub 32mb 8mb 60
```

一般情况下，对于普通客户端，client-output-buffer 是不设限制的，因为客户端每发送一个请求，会等到响应结果后，再进行下一个请求，除非`bigkey`，一般不会积压在缓冲区。对于用作 Pub/Sub 和 slave 的客户端，server 会主动把数据推送给他们，故需要设置 client-output-buffer 的限制。

### 复制缓冲区

主从全量复制过程中，主节点向从节点传输RDB文件的同时，将接收到的客户端写命令保存在复制缓冲区中，等RDB文件传输完成后，再发生给从节点，主节点会为每个从节点维护一个复制缓冲区。

所以复制缓冲区本质上还是一个用于从节点使用的输出缓冲区，发生溢出也会直接关闭。设置缓冲区大小见上面(slave)。

### 复制积压缓冲区

主节点在接收到命令同步给从节点时，会同时写到复制积压缓冲区，用于在网络断开恢复后，将断连期间的写命令，增量同步给从节点。复制积压缓冲区是一个有限的环形缓冲区，写满后会覆盖之前的旧数据，如果从节点还没有同步这些旧命令，会导致主从节点间重新开始全量复制。对比复制缓冲区，发送出去的数据会清除，复制积压缓冲区只会覆盖旧数据，仍然保留了最新的命令。通过`repl_backlog_size`配置复制积压缓冲区的大小。

## 淘汰机制

```shell
config set maxmemory 4gb
```

设置Redis使用的最大内存，缓存写满后会触发缓存淘汰策略。

| 策略            | 说明                                               |
| --------------- | -------------------------------------------------- |
| noeviction      | 不进行数据淘汰(默认)                               |
| volatile-random | 针对设置了过期时间的键值对，随机删除               |
| volatile-ttl    | 针对设置了过期时间的键值对，根据过期的先后进行删除 |
| volatile-lru    | 针对设置了过期时间的键值对，删除最近最没使用的     |
| volatile-lfu    | 针对设置了过期时间的键值对，删除最近最少使用的     |
| allkeys-random  | 从所有键值对中随机选择删除                         |
| allkeys-lru     | 从所有键值对中删除最近最没有使用的                 |
| allkeys-lfu     | 从所有键值对中删除最近最少使用的                   |

Redis默认记录每个数据的最近一次访问时间戳，保存在RedisObject的lru字段，在决定淘汰数据时，第一次会随机选出N个数据，比较这N个数据的lru字段，把lru最小的删除。通过配置参数`CONFIG SET maxmemory-samples 100`设置选出的样本数N。再次淘汰数据时，Redis需要挑选数据进入第一次淘汰时创建的候选集合，能进入候选集合的数据lru必须小于集合中最小的lrj值。当候选数据个数达到`maxmemory-samples`时，把lru最小的淘汰出去。

## 缓存异常

### 缓存雪崩

缓存雪崩是指大量的应用请求无法在Redis缓存中进行处理，大量的请求送到数据库层，导致数据库压力激增。一般是由于缓存中大量数据同时过期，导致大量请求无法得到处理，或者Redis缓存实例发生故障宕机了，无法处理请求。

针对大量缓存同时过期，可以在给缓存设置过期时间时，加一个较小的随机数，避免大量数据同时过期；

处理方式一般是服务降级：非核心业务，暂时停止从缓存中查询，而是返回预定义信息或者错误信息；核心业务，仍然运行查询缓存，缓存缺失则访问数据库。

如果是服务宕机，为了防止引发连锁的数据库雪崩，甚至整个系统的崩溃，在业务系统中实现服务熔断或请求限流。

### 缓存击穿

某个热点数据失效的场景。相比缓存雪崩，缓存击穿的数据量要小很多。

处理方式是针对热点数据，不设置过期时间。

### 缓存穿透

要访问的数据既不在Redis缓存中，也不在数据库中，导致每次请求都要访问数据库。处理方式是缓存空值或默认值，这样避免请求到底数据库；或者使用布隆过滤器判断数据是否存在，如果不存在就不用访问数据库了；或者针对恶意请求访问不存在的数据，在前端对请求进行过滤。

### 缓存污染

留存在缓存中，但是访问次数很少，或者以后不会再被访问的数据。使用LFU算法的淘汰策略能有效解决缓存污染的问题。Redis在实现LFU策略时，将`lru`字段拆分为两部分：前16bit的时间戳(分钟级别)和后8bit的访问次数。

8bit的访问次数理论最大只有255，但是Redis的实现是一个相对值。计算 $ 1 / (counter * lfu\_log\_factor + 1) $ 得到的值和一个$(0, 1)$区间的随机数`r`比较，比 `r`大`counter++`，所以计数器增加是一个概率事件，而且计数器越大，这个概率越低。同时，Redis还加入了计数器衰减的机制，首先计算当前时间和`lru`时间戳的分钟差值，用差值除以`lfu_decay_time`得到计数器要衰减的值。

> 用`lfu_log_factor`限制计数器增长的速度，用`lfu_decay_time`控制计数器衰减的速度。
>
> 这里猜测可能有个问题，前16bit表示的`ldt`上次衰减时间计算为` ((unixtime / 60) & 65535)`，最大时间间隔是65535分钟，约45天。若刚好65535分钟后，访问触发衰减时，衰减值` (now >= ldt ? (now - ldt) : (65535 - ldt + now)) / lfu_decay_time ​` 很小，导致本该被淘汰的值继续留存下来。

## 原子操作

### 单命令操作

Redis是单线程处理客户端命令的，所以对于单条命令是原子操作。如果多个操作可以用一条命令实现相同的效果，那么就可以用这个单命令来实现原子操作。比如`DECR <KEY>`用来替代"GET, -1, SET"的操作。

### LUA脚本

对于较为复杂的操作，无法用单条命令实现的，用LUA脚本来保证原子操作。

```lua
local current
current = redis.call("incr", KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire", KEYS[1], 60)
end
```

然后调用脚本`redis-cli --eval xxx.lua keys, args`

### 分布式锁

**基于单个Redis节点的分布式锁**

加锁：`SET lock_key <RANDOM_STR> NX EX 5`，单命令保证原子性，`NX`只有键不存在时才能创建成功，`EX`设置过期时间5s，随机字符串避免被其他客户端删除。

解锁：`redis-cli --eval unlock.lua lock_key, <RANDOM_STR>`，lua脚本保证原子性，参数包含加锁时的随机字符串，避免锁的误删。

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

**基于多个Redis节点的高可靠分布式锁**

分布式锁算法Redlock：

1. 客户端获取当前时间
2. 客户端依次向N个Redis实例执行加锁操作，加锁操作的超时时间远小于锁的有效时间
3. 客户端完成向所有Redis实例的加锁操作，计算整个过程的耗时

同时满足超过半数($N/2 + 1$)的实例加锁超过和总耗时不超过锁的有效时间才算加锁超过。如果不成功，客户端就向所有Redis实例发起释放锁的操作。

## 事务

事务的ACID特性：

+ Atomicity原子性
+ Consistency一致性
+ Isolation隔离性
+ Durability持久性

Redis通过`MULTI`和`EXEC`命令提供事务的支持。

通过这两个命令提交的事务命令要么都**执行**，要么都**不执行**。强调的是执行，而不管成不成功，如果：命令有语法错误，命令入队时会报错，则会放弃，都不执行，保证原子性；如果非语法错误，但是命令执行时报错，全部执行，保证原子性；如果使用`DISCARD`放弃事务，则全部不执行，保证原子性；如果Redis实例故障，在开启AOF日志的前提下，使用工具`redis-check-aof`可以检测未完成的事务，恢复时不执行，保证原子性。

Redis 提供了 `WATCH`命令来保证事务的隔离性，使用`WATCH`对一个或多个key 进行监控，在`EXEC`之前，如果有其他客户端修改了监控的 key，事务会被放弃。

Redis 在两次 RDB 之间，或者 AOF 刷盘前宕机，是保证不了持久性的。