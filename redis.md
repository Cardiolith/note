# Redis Note

> #### Linux Redis 安装

```shell
wget http://download.redis.io/releases/redis-5.0.7.tar.gz
tar -xzvf redis-5.0.7.tar.gz
mv redis-5.0.7.tar.gz redis
cd redis
make 
make install RPEFIX=/home/**/redis
```

 

> #### Redis 命令

- key

  ```shell
  DEL key [key] 				// 删除key,返回删除的数量
  
  DUMP key					// 序列化key,返回序列化后的值
  
  EXISTS key					// 检查key是否存在，存在返回1，否则返回0
  
  EXPIRE key seconds			// 设置key的生存时间
  
  TTL	key 					// 返回key的剩余生存时间
  
  PERSIST key					// 移除key的生存时间限制
  
  RANDOMKEY 					// 随即返回一个key
  
  RENAME key newkey			// 重命名key, 当key不存在或key和newkey相同时返回错误。会覆盖
  
  RENAMENX key newkey			// 仅当newkey不存在时,重命名
  
  RESTORE key value			// 反序列化value, 并与key关联
  
  SORT key [DESC]				// 返回对list, set, zset 排序后的结果
  
  SORT key ALPHA				// 按字符串排序, 默认数字排序
  
  SORT key LIMIT offset count	// 指定排序的偏移量和返回数量
  
  TYPE key 					// 返回指定key的类型
  
  SCAN cursor	[match] [count]	// 迭代数据库KEY, 返回下次迭代的游标和keys, 0表示迭代已结束
  
  SSCAN key cursor 			// 迭代集合元素
  
  HSCAN key cursor 			// 迭代每一个键值对
  
  ZSCAN key cursor			// 迭代集合元素和分值
  ```

- string 

  ```
  APPEND key value			// 将value追加到原来的值末尾,如果key不存在则等价于set
  
  ```
  
- list

  ```shell
  rpush list-key item		// 返回当前list长度
  
  lrange list-key 0 -1	// 取list index 范围内的item
  
  lindex list-key 1		// 取list指定index的item
  
  lpop list-key			// 从list弹出一个元素，此元素将不再存在于list
  
  ```

- set

  ```shell
  sadd set-key item		// 集合添加元素,返回0表示已经存在，1表示插入成功
  
  smembers set-key 		// 返回set所有元素
  
  sismember set-key item	// 检查一个元素是否在集合内
  
  srem set-key [member]	// 移除set内的元素，返回移除的数量
  
  
  ```

- hash

  ```shell
  hset hash-key sub-key1 value1	// 添加键到hash，返回0/1表示键是否已存在
  
  hgetall hash-key				// 返回hash所有键值对
  
  hdel hash-key sub-key			// 删除hash某个简直
  
  hget hash-key sub-key			// 获取hash某个键值
  ```

- zset

  ```
  zadd zset-key score member1		// 向有序集合添加元素，返回新添加元素的数量
  
  zrange zset-key 0 -1 withscores // 获取zset多个元素，并排序
  
  zrangebyscore zset-key 0 800 	// 根据分值返回部分元素
  
  zrem zset-key member1			// 移除元素
  ```



### Redis 架构

#### Redis维度与主线

![img](https://static001.geekbang.org/resource/image/79/e7/79da7093ed998a99d9abe91e610b74e7.jpg?wh=2001*1126)

#### Redis问题

![img](https://static001.geekbang.org/resource/image/70/b4/70a5bc1ddc9e3579a2fcb8a5d44118b4.jpeg?wh=2048*1536)

#### Redis模块

一个键值数据库包括了访问框架、索引模块、操作模块和存储模块

![img](https://static001.geekbang.org/resource/image/30/44/30e0e0eb0b475e6082dd14e63c13ed44.jpg)

#### Redis单线程

redis单线程是指网络IO与key-value读写是由一个线程完成的。其他功能比如持久化、异步删除、数据同步事由其他线程完成的。

多线程一般需要引入同步原语来保护共享资源的并发访问。

##### Redis为什么快？

大部分操作在内存中进行，高效的数据结构。另一方面，采用多路复用机制，使其可以在网络IO操作中能并发处理大量的客户端请求。

##### Redis基本IO模型

![img](https://static001.geekbang.org/resource/image/e1/c9/e18499ab244e4428a0e60b4da6575bc9.jpg)

accept和recv步骤可能因为客户端的原因导致阻塞，不过 socket 网络模型支持非阻塞模型。

![img](https://static001.geekbang.org/resource/image/1c/4a/1ccc62ab3eb2a63c4965027b4248f34a.jpg)

针对监听套接字，我们可以设置非阻塞模式：当 Redis 调用 accept() 但一直未有连接请求到达时，Redis 线程可以返回处理其他操作，而不用一直等待。但是，你要注意的是，调用 accept() 时，已经存在监听套接字了。



##### 基于多路复用的高性能 I/O 模型

Linux 中的 IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 **select/epoll** 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，**同时存在多个监听套接字和已连接套接字**。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果。

下图就是基于多路复用的 Redis IO 模型。图中的多个 FD 就是刚才所说的多个套接字。Redis 网络框架调用 epoll 机制，让内核监听这些套接字。此时，Redis 线程不会阻塞在某一个特定的监听或已连接套接字上，也就是说，不会阻塞在某一个特定的客户端请求处理上。正因为此，Redis 可以同时和多个客户端连接并处理请求，从而提升并发性。

![img](https://static001.geekbang.org/resource/image/00/ea/00ff790d4f6225aaeeebba34a71d8bea.jpg)



为了在请求到达时能通知到 Redis 线程，select/epoll 提供了基于事件的回调机制，即针对不同事件的发生，调用相应的处理函数。那么，回调机制是怎么工作的呢？其实，select/epoll 一旦监测到 FD 上有请求到达时，就会触发相应的事件。



这些事件会被放进一个事件队列，Redis 单线程对该事件队列不断进行处理。这样一来，Redis 无需一直轮询是否有请求实际发生，这就可以避免造成 CPU 资源浪费。同时，Redis 在对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。因为 Redis 一直在对事件队列进行处理，所以能及时响应客户端请求，提升 Redis 的响应性能。



##### Redis单线程处理IO请求性能瓶颈主要包括2个方面：

1、任意一个请求在server中一旦发生耗时，都会影响整个server的性能，也就是说后面的请求都要等前面这个耗时请求处理完成，自己才能被处理到。耗时的操作包括以下几种：
a、操作bigkey：写入一个bigkey在分配内存时需要消耗更多的时间，同样，删除bigkey释放内存同样会产生耗时；
b、使用复杂度过高的命令：例如SORT/SUNION/ZUNIONSTORE，或者O(N)命令，但是N很大，例如lrange key 0 -1一次查询全量数据；
c、大量key集中过期：Redis的过期机制也是在主线程中执行的，大量key集中过期会导致处理一个请求时，耗时都在删除过期key，耗时变长；
d、淘汰策略：淘汰策略也是在主线程执行的，当内存超过Redis内存上限后，每次写入都需要淘汰一些key，也会造成耗时变长；
e、AOF刷盘开启always机制：每次写入都需要把这个操作刷到磁盘，写磁盘的速度远比写内存慢，会拖慢Redis的性能；
f、主从全量同步生成RDB：虽然采用fork子进程生成数据快照，但fork这一瞬间也是会阻塞整个线程的，实例越大，阻塞时间越久；
2、并发量非常大时，单线程读写客户端IO数据存在性能瓶颈，虽然采用IO多路复用机制，但是读写客户端数据依旧是同步IO，只能单线程依次读取客户端的数据，无法利用到CPU多核。

针对问题1，一方面需要业务人员去规避，一方面Redis在4.0推出了lazy-free机制，把bigkey释放内存的耗时操作放在了异步线程中执行，降低对主线程的影响。

针对问题2，Redis在6.0推出了多线程，可以在高并发场景下利用CPU多核多线程读写客户端数据，进一步提升server性能，当然，只是针对客户端的读写是并行的，每个命令的真正操作依旧是单线程的。

![img](https://static001.geekbang.org/resource/image/4d/9f/4d120bee623642e75fdf1c0700623a9f.jpg)

#### Redis数据恢复（AOF / RDB）

写前日志(Write Ahead Log)，实际写之前把修改的记录到日志文件中。

AOF是写后日志，Redis执行完命令，把数据写入内存，然后才记录日志。

传统数据库的日志，记录的是修改后的数据，而AOF记录的是Redis收到的每一条命令。

我们以 Redis 收到“set testkey testvalue”命令后记录的日志为例，看看 AOF 日志的内容。其中，“*3”表示当前命令有三个部分，每部分都是由“$+数字”开头，后面紧跟着具体的命令、键或值。这里，“数字”表示这部分中的命令、键或值一共有多少字节。例如，“$3 set”表示这部分有 3 个字节，也就是“set”命令。

AOF还有一个好处是：不会阻塞当前写操作。

###### AOF风险：

- 如果执行完一个命令后服务宕机，那么就有丢失的风险。
- 虽然不会对当前命令阻塞，但可能会对下一个操作带来阻塞风险，因为AOF也是在主线程中进行的。

###### 写回策略：

- Always: 同步写回。
- EverySec: 每秒写回，每个写命令执行完，只是把日志写回到AOF的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘。
- No: 由操作系统控制写回，每个写命令执行完，只是把日志写入到缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

![img](https://static001.geekbang.org/resource/image/72/f8/72f547f18dbac788c7d11yy167d7ebf8.jpg)



**性能问题：**

- 文件系统本身对文件大小限制。
- 文件过大，追加内容效率变低。
- 宕机恢复过程缓慢。

###### AOF重写机制：

旧日志文件中的多条命令，在重写后的新日志变成了一条命令。

在重写的时候，是根据这个键值的最新状态，为它生成对应的写入命令。

AOF重写过程由后台子进程bgrewriteaof来完成的。

我把重写的过程总结为“一个拷贝，两处日志”。

“一个拷贝”就是指，每次执行重写时，**主线程 fork 出后台的 bgrewriteaof 子进程**。此时，fork 会把主线程的内存拷贝一份给 bgrewriteaof 子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof 子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志。

“两处日志”又是什么呢？

因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的 AOF 日志，Redis 会把这个操作写到它的缓冲区。这样一来，即使宕机了，这个 AOF 日志的操作仍然是齐全的，可以用于恢复。而第二处日志，就是指新的 AOF 重写日志。这个操作也会被写到重写日志的缓冲区。这样，重写日志也不会丢失最新的操作。等到拷贝数据的所有操作记录重写完成后，重写日志记录的这些最新操作也会写入新的 AOF 文件，以保证数据库最新状态的记录。此时，我们就可以用新的 AOF 文件替代旧文件了。

![img](https://static001.geekbang.org/resource/image/6b/e8/6b054eb1aed0734bd81ddab9a31d0be8.jpg)

###### RDB（Redis DataBase）:	全量快照

Redis 提供两个命令生成RDB文件，分别是save和bgsave：

- save:	在主线程中执行，会导致阻塞
- bgsave:     创建一个子线程，专门用于写入RDB文件

你可能会想到，可以用 bgsave 避免阻塞啊。这里我就要说到一个常见的误区了，避免阻塞和正常处理写操作并不是一回事。此时，主线程的确没有阻塞，可以正常接收请求，但是，**为了保证快照完整性，它只能处理读操作，因为不能修改正在执行快照的数据**。

###### 写时复制技术（Copy-On-Write, COW）

在执行快照的同时，正常处理写操作。

bgsave 子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据。bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 RDB 文件。

此时，如果主线程对这些数据也都是读操作（例如图中的键值对 A），那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对 C），那么，这块数据就会被复制一份，生成该数据的副本（键值对 C’）。然后，主线程在这个数据副本上进行修改。同时，bgsave 子进程可以继续把原来的数据（键值对 C）写入 RDB 文件。

![img](https://static001.geekbang.org/resource/image/a2/58/a2e5a3571e200cb771ed8a1cd14d5558.jpg)

###### 快照机制：

我们先在 T0 时刻做了一次快照，然后又在 T0+t 时刻做了一次快照，在这期间，数据块 5 和 9 被修改了。如果在 t 这段时间内，机器宕机了，那么，只能按照 T0 时刻的快照进行恢复。此时，数据块 5 和 9 的修改值因为没有快照记录，就无法恢复了。

![img](https://static001.geekbang.org/resource/image/71/ab/711c873a61bafde79b25c110735289ab.jpg)



如果频繁地执行全量快照，也会带来两方面的开销。

一方面，频繁将全量数据写入磁盘，会给磁盘带来很大压力，多个快照竞争有限的磁盘带宽，前一个快照还没有做完，后一个又开始做了，容易造成恶性循环。

另一方面，bgsave 子进程需要通过 fork 操作从主线程创建出来。虽然，子进程在创建后不会再阻塞主线程，但是，fork 这个创建过程本身会阻塞主线程，而且主线程的内存越大，阻塞时间越长。如果频繁 fork 出 bgsave 子进程，这就会频繁阻塞主线程了（所以，在 Redis 中如果有一个 bgsave 在运行，就不会再启动第二个 bgsave 子进程）。

###### 增量快照：

指做了一次全量快照之后，后续的快照针对修改的数据进行快照记录。

我们只需要将被修改的数据写入快照文件就行。但是，这么做的前提是，我们需要记住哪些数据被修改了。你可不要小瞧这个“记住”功能，它需要我们使用额外的元数据信息去记录哪些数据被修改了，这会带来额外的空间开销问题。



#### Redis 数据同步

##### 主从库模式：

![img](https://static001.geekbang.org/resource/image/80/2f/809d6707404731f7e493b832aa573a2f.jpg)



##### 主从库如何同步？

第一次同步:

![img](https://static001.geekbang.org/resource/image/63/a1/63d18fd41efc9635e7e9105ce1c33da1.jpg)

FULLRESYNC 响应表示第一次复制采用的全量复制，也就是说，主库会把当前所有的数据都复制给从库。

在主库将数据同步给从库的过程中，主库不会被阻塞，仍然可以正常接收请求。否则，Redis 的服务就被中断了。但是，这些请求中的写操作并没有记录到刚刚生成的 RDB 文件中。为了保证主从库的数据一致性，**主库会在内存中用专门的 replication buffer**，记录 RDB 文件生成后收到的所有写操作。

最后，也就是第三个阶段，主库会把第二阶段执行过程中新收到的写命令，再发送给从库。具体的操作是，当主库完成 RDB 文件发送后，就会把此时 replication buffer 中的修改操作发给从库，从库再重新执行这些操作。这样一来，主从库就实现同步了。

一旦主从库完成了全量复制，它们之间就会一直维护一个网络连接，主库会通过这个连接将后续陆续收到的命令操作再同步给从库，这个过程也称为基于长连接的命令传播，可以避免频繁建立连接的开销。

##### 主从网络断开处理：

Redis 2.8之前，主从之间就会进行一次全量复制。

Redis 2.8之后，会基于增量复制，只会把主从库网络断开期间主库收到的命令，同步给从库。

当主从库断连后，主库会把断连期间收到的写操作命令，写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。

repl_backlog_buffer 是一个环形缓冲区，主库会记录自己写到的位置，从库则会记录自己已经读到的位置。

刚开始的时候，主库和从库的写读位置在一起，这算是它们的起始位置。随着主库不断接收新的写操作，它在缓冲区中的写位置会逐步偏离起始位置，我们通常用偏移量来衡量这个偏移距离的大小，对主库来说，对应的偏移量就是 master_repl_offset。主库接收的新写操作越多，这个值就会越大。

同样，从库在复制完写操作命令后，它在缓冲区中的读位置也开始逐步偏移刚才的起始位置，此时，从库已复制的偏移量 slave_repl_offset 也在不断增加。正常情况下，这两个偏移量基本相等。

![img](https://static001.geekbang.org/resource/image/13/37/13f26570a1b90549e6171ea24554b737.jpg)

主从库的连接恢复之后，从库首先会给主库发送 psync 命令，并把自己当前的 slave_repl_offset 发给主库，主库会判断自己的 master_repl_offset 和 slave_repl_offset 之间的差距。在网络断连阶段，主库可能会收到新的写操作命令，所以，一般来说，master_repl_offset 会大于 slave_repl_offset。此时，主库只用把 master_repl_offset 和 slave_repl_offset 之间的命令操作同步给从库就行。

因为 repl_backlog_buffer 是一个环形缓冲区，所以在缓冲区写满后，主库会继续写入，此时，就会覆盖掉之前写入的操作。如果从库的读取速度比较慢，就有可能导致从库还未读取的操作被主库新写的操作覆盖了，这会导致主从库间的数据不一致。

一般而言，我们可以调整 repl_backlog_size 这个参数。这个参数和所需的缓冲空间大小有关。缓冲空间的计算公式是：缓冲空间大小 = 主库写入命令速度 * 操作大小 - 主从库间网络传输命令速度 * 操作大小。在实际应用中，考虑到可能存在一些突发的请求压力，我们通常需要把这个缓冲空间扩大一倍，即 repl_backlog_size = 缓冲空间大小 * 2，这也就是 repl_backlog_size 的最终值。



#### Redis 哨兵机制

哨兵是主从库模式下的特殊的Redis进程，负责监控、选主（选择主库）和通知。

- 监控：向所有主从库发送PING，如果从库没有在规定时间内响应哨兵的PING命令，就会把它标记为‘下线状态’，如果是主库，哨兵就会判定主库下线，然后开始自动切换主库的流程。
- 选主：选主，主库挂了以后，就会根据一定规则选择一个从库实例，将它作为主库。
- 通知：把新主库的信息发送给各个从库，让它们执行replicaof命令，和新主库建立连接，并进行数据复制。

##### 防止误判：

哨兵机制一般也采用多实例的集群模型进行部署，避免单个哨兵因网络原因误判主库下线。

##### 如何选定新主库：

筛选+打分：

![img](https://static001.geekbang.org/resource/image/f2/4c/f2e9b8830db46d959daa6a39fbf4a14c.jpg)

只要在某一轮中，有从库得分最高，那么它就是主库了，选主过程到此结束。如果没有出现得分最高的从库，那么就继续进行下一轮。

- 第一轮：优化级最高的从库得分高。可以通过slave-priority配置项，给不同的从库设置不同优先级。
- 第二轮：和旧主库同步程度最接近的从库得分高。
- 第三轮：ID号小的从库得分高。

#### 哨兵集群

##### 基于 pub/sub 机制的哨兵集群组成

如果你部署过哨兵集群的话就会知道，在配置哨兵的信息时，我们只需要用到下面的这个配置项，设置主库的 IP 和端口，并没有配置其他哨兵的连接信息。

哨兵实例之间可以相互发现，要归功于 Redis 提供的 pub/sub 机制，也就是发布 / 订阅机制。

哨兵只要和主库建立起了连接，就可以在主库上发布消息了，比如说发布它自己的连接信息（IP 和端口）。同时，它也可以从主库上订阅消息，获得其他哨兵发布的连接信息。当多个哨兵实例都在主库上做了发布和订阅操作后，它们之间就能知道彼此的 IP 地址和端口。



哨兵除了彼此之间建立起连接形成集群外，还需要和从库建立连接。这是因为，在哨兵的监控任务中，它需要对主从库都进行心跳判断，而且在主从库切换完成后，它还需要通知从库，让它们和新主库进行同步。



从本质上说，哨兵就是一个运行在特定模式下的 Redis 实例，只不过它并不服务请求操作，只是完成监控、选主和通知的任务。所以，每个哨兵实例也提供 pub/sub 机制，客户端可以从哨兵订阅消息。哨兵提供的消息订阅频道有很多，不同频道包含了主从库切换过程中的不同关键事件。



#### Redis 切片集群

具体来说，Redis Cluster 方案采用哈希槽（Hash Slot，接下来我会直接称之为 Slot），来处理数据和实例之间的映射关系。在 Redis Cluster 方案中，一个切片集群共有 16384 个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈希槽中。

具体的映射过程分为两大步：首先根据键值对的 key，按照CRC16 算法计算一个 16 bit 的值；然后，再用这个 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。

Redis 实例会把自己的哈希槽信息发给和它相连接的其它实例，来完成哈希槽分配信息的扩散。当实例之间相互连接后，每个实例就有所有哈希槽的映射关系了。

但是，在集群中，实例和哈希槽的对应关系并不是一成不变的，最常见的变化有两个：

- 在集群中，实例有新增或删除，Redis 需要重新分配哈希槽；
- 为了负载均衡，Redis 需要把哈希槽在所有实例上重新分布一遍。

![img](https://static001.geekbang.org/resource/image/35/09/350abedefcdbc39d6a8a8f1874eb0809.jpg)

Redis Cluster 方案提供了一种重定向机制，所谓的“重定向”，就是指，客户端给一个实例发送数据读写操作时，这个实例上并没有相应的数据，客户端要再给一个新实例发送操作命令。

那客户端又是怎么知道重定向时的新实例的访问地址呢？当客户端把一个键值对的操作请求发给一个实例时，如果这个实例上并没有这个键值对映射的哈希槽，那么，这个实例就会给客户端返回下面的 MOVED 命令响应结果，这个结果中就包含了新实例的访问地址。



在上图中，当客户端给实例 2 发送命令时，Slot 2 中的数据已经全部迁移到了实例 3。在实际应用时，如果 Slot 2 中的数据比较多，就可能会出现一种情况：客户端向实例 2 发送请求，但此时，Slot 2 中的数据只有一部分迁移到了实例 3，还有部分数据没有迁移。在这种迁移部分完成的情况下，客户端就会收到一条 ASK 报错信息，如下所示：

```shell
GET hello:key
(error) ASK 13320 172.16.19.5:6379
```

这个结果中的 ASK 命令就表示，客户端请求的键值对所在的哈希槽 13320，在 172.16.19.5 这个实例上，但是这个哈希槽正在迁移。此时，客户端需要先给 172.16.19.5 这个实例发送一个 ASKING 命令。这个命令的意思是，让这个实例允许执行客户端接下来发送的命令。然后，客户端再向这个实例发送 GET 命令，以读取数据。



![img](https://static001.geekbang.org/resource/image/67/36/67e77bea2568a4f0997c1853d9c60036.jpg)

#### Redis 数据结构

![img](https://static001.geekbang.org/resource/image/82/01/8219f7yy651e566d47cc9f661b399f01.jpg)

Redis 使用一张全局哈希表，用来保存所有键值对。

![img](https://static001.geekbang.org/resource/image/1c/5f/1cc8eaed5d1ca4e3cdbaa5a3d48dfb5f.jpg)

Hash表，其实就是一个数组，数组每个元素称为一个Hash桶。

查找键值对的时间复杂度O(1)。

**Hash冲突解决方式**：链式Hash，同一个Hash桶的元素用链表保存。

Rehash：当hash冲突越来越多时，查找效率降低。rehash即是增加现有的hash桶数量，减少冲突的发生。

```
Rehash steps:

1. init hashtable1 hashtable2

2. begin rehash

3. malloc hashtable2 size twice than hashtable1 size

4. copy hashtable1 mem to hashtable2 mem

5. free hashtable1
```

***渐进式rehash***：一次性执行rehash很可能导致线程阻塞。解决办法：将rehash的copy过程，分散到多次请求中。

![img](https://static001.geekbang.org/resource/image/73/0c/73fb212d0b0928d96a0d7d6ayy76da0c.jpg)



> Redis 时间复杂度

![img](https://static001.geekbang.org/resource/image/95/a0/9587e483f6ea82f560ff10484aaca4a0.jpg)

```
压缩列表(ZipList):
zlbytes: 存储一个无符号整数，固定四个字节长度，用于存储压缩列表所占用的字节，当重新分配内存的时候使用，不需要遍历整个列表来计算内存大小
zltail: 存储一个无符号整数，固定四个字节长度，代表指向列表尾部的偏移量，偏移量是指压缩列表的起始位置到指定列表节点的起始位置的距离
zllen: 压缩列表包含的节点个数，固定两个字节长度，源码中指出当节点个数大于2^16-2个数的时候，该值将无效，此时需要遍历列表来计算列表节点的个数
entryN: 列表节点区域，长度不定，由列表节点紧挨着组成
zlend: 一字节长度固定值为255，用于表示列表结束
```

上面介绍了压缩列表的总体内存布局，对于初entryX区域以外的四个区域的长度都是固定的，下面再看看entryX区域的编码情况。

每个列表节点由三部分组成：

![img](https://pic2.zhimg.com/80/v2-00fcbc58da96e178e76bff6487c2c1a5_1440w.jpg)

```
previous length: 用于存储上一个节点的长度
encoding: 保存的是节点的content的内容类型以及长度
content: 用于保存节点的内容，节点内容类型和长度由encoding决定
```

**跳表**：有序链表的优化

![img](https://static001.geekbang.org/resource/image/1e/b4/1eca7135d38de2yy16681c2bbc4f3fb4.jpg)



![img](https://static001.geekbang.org/resource/image/fb/f0/fb7e3612ddee8a0ea49b7c40673a0cf0.jpg)



### Redis 实战

#### String 类型内存开销大？

当你保存 64 位有符号整数时，String 类型会把它保存为一个 8 字节的 Long 类型整数，这种保存方式通常也叫作 int 编码方式。

但是，当你保存的数据中包含字符时，String 类型就会用简单动态字符串（Simple Dynamic String，SDS）结构体来保存，如下图所示：

![img](https://static001.geekbang.org/resource/image/37/57/37c6a8d5abd65906368e7c4a6b938657.jpg)

- buf：字节数组，保存实际数据。为了表示字节数组的结束，Redis 会自动在数组最后加一个“\0”，这就会额外占用 1 个字节的开销。

- len：占 4 个字节，表示 buf 的已用长度。

- alloc：也占个 4 字节，表示 buf 的实际分配长度，一般大于 len。

另外，对于 String 类型来说，除了 SDS 的额外开销，还有一个来自于 RedisObject 结构体的开销。

因为 Redis 的数据类型有很多，而且，不同数据类型都有些相同的元数据要记录（比如最后一次访问的时间、被引用的次数等），所以，Redis 会用一个 RedisObject 结构体来统一记录这些元数据，同时指向实际数据。

一个 RedisObject 包含了 8 字节的元数据和一个 8 字节指针，这个指针再进一步指向具体数据类型的实际数据所在，例如指向 String 类型的 SDS 结构所在的内存地址。

![img](https://static001.geekbang.org/resource/image/34/57/3409948e9d3e8aa5cd7cafb9b66c2857.jpg)

为了节省内存空间，Redis 还对 Long 类型整数和 SDS 的内存布局做了专门的设计。

一方面，当保存的是 Long 类型整数时，RedisObject 中的指针就直接赋值为整数数据了，这样就不用额外的指针再指向整数了，节省了指针的空间开销。

另一方面，当保存的是字符串数据，并且字符串小于等于 44 字节时，RedisObject 中的元数据、指针和 SDS 是一块连续的内存区域，这样就可以避免内存碎片。这种布局方式也被称为 embstr 编码方式。

当然，当字符串大于 44 字节时，SDS 的数据量就开始变多了，Redis 就不再把 SDS 和 RedisObject 布局在一起了，而是会给 SDS 分配独立的空间，并用指针指向 SDS 结构。这种布局方式被称为 raw 编码模式。

![img](https://static001.geekbang.org/resource/image/ce/e3/ce83d1346c9642fdbbf5ffbe701bfbe3.jpg)



Redis 会使用一个全局哈希表保存所有键值对，哈希表的每一项是一个 dictEntry 的结构体，用来指向一个键值对。dictEntry 结构中有三个 8 字节的指针，分别指向 key、value 以及下一个 dictEntry，三个指针共 24 字节，如下图所示：

![img](https://static001.geekbang.org/resource/image/b6/e7/b6cbc5161388fdf4c9b49f3802ef53e7.jpg)

jemalloc 在分配内存时，会根据我们申请的字节数 N，找一个比 N 大，但是最接近 N 的 2 的幂次数作为分配的空间，这样可以减少频繁分配的次数。

举个例子。如果你申请 6 字节空间，jemalloc 实际会分配 8 字节空间；如果你申请 24 字节空间，jemalloc 则会分配 32 字节。所以，在我们刚刚说的场景里，dictEntry 结构就占用了 32 字节。
