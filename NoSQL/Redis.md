##### 为什么使用Redis?

1. ###### Redis支持丰富的数据类型

* redis提供5中数据类型，list, string, hash, set, soted set
* 各种各样的问题都可以很自然的映射到这些数据结构上，因此redis可以实现很多有用的功能：
  * 用list做FIFO双向链表，实现一个轻量级的高性能消息队列服务
  * 对存入的Key-Value设置expire时间作为会话session

2. ###### Redis很快

* Redis有多快？

```shell
λ redis-benchmark.exe -q
PING_INLINE: 115473.45 requests per second
PING_BULK: 125944.58 requests per second
SET: 117785.63 requests per second
GET: 125786.16 requests per second
INCR: 123152.71 requests per second
LPUSH: 115740.73 requests per second
RPUSH: 119760.48 requests per second
LPOP: 116279.07 requests per second
RPOP: 119189.52 requests per second
SADD: 116959.06 requests per second
SPOP: 124533.01 requests per second
LPUSH (needed to benchmark LRANGE): 119331.74 requests per second
LRANGE_100 (first 100 elements): 34270.05 requests per second
LRANGE_300 (first 300 elements): 15610.37 requests per second
LRANGE_500 (first 450 elements): 11305.82 requests per second
LRANGE_600 (first 600 elements): 8690.36 requests per second
MSET (10 keys): 82101.80 requests per second
```
* Redis为什么这么快？
  * **redis是一个基于内存的数据库，存取操作都在内存中进行，而绝大部分请求都是纯粹的内存操作**
  * **redis采用单进程单线程模型，保证每个操作的原子性，避免不必要的上下文切换和竞争条件**
    * 单线程是指网络请求模块使用一个线程处理所有网络请求，所以不需考虑并发安全性
      * Redis客户端对服务端的每次调用都经历了发送命令，执行命令，返回结果三个过程。其中执行命令阶段，由于Redis是单线程来处理命令的，所有每一条到达服务端的命令不会立刻执行，所有的命令都会进入一个队列中，然后逐个被执行。并且多个客户端发送的命令的执行顺序是不确定的。但是可以确定的是不会有两条命令被同时执行，不会产生并发问题，这就是Redis的单线程基本模型 。
    * 采用单线程，是因为CPU不是redis的瓶颈，redis的瓶颈最有可能是内存和网络带宽。
    * 单线程无法发挥多核CPU的优势，不过可以通过单机开多个redis实例或者redis集群来完善

  * **采用非阻塞IO和IO多路复用**
    * 非阻塞IO是5种IO模型中的一种，是指在线程的执行过程中，当产生一个IO操作（系统调用）时，线程不会被阻塞，而是使用一个循环来不停的测试IO操作是否已经完成
    * 非阻塞IO因为要用一个死循环来轮询，极为耗费CPU资源，因此常需要和IO多路复用配合使用
    * IO多路复用，它是指内核一旦发现进程(线程)指定的一个或多个IO条件(文件描述符fd)准备就绪，它就会通知该进程(线程)。IO多路复用有三种机制：select、poll、epoll，具体的讲就是，进程(线程)阻塞于select/poll/epoll系统调用，等待多个IO中的任一个就绪，select/poll/epoll调用返回，唤醒进程(线程)通知相应的IO可以读，这样就避免了非阻塞IO中轮询等待的问题。
      * select/poll顺序扫描fd是否就绪，支持的fd也有限
      * epoll基于事件驱动的方式代替顺序扫描，因此效率更高，基于epoll的IO多路复用模型也叫事件驱动模型
      * redis内部实现采用的就是epoll和自身的事件处理模型，epoll中的读、写、关闭、连接都转化成事件，然后利用epoll的多路复用特性，不在网络IO上浪费时间
      * 所谓IO多路复用，其实就是指IO复用模型能够阻塞多个IO操作，即一个线程就能够处理多个IO，因此，redis能以单线程运行的同时服务成千上万个文件描述符，避免了由于多线程的引入导致代码实现复杂度的提升，减少了出错的可能性。

3. ###### Redis支持持久化

- Redis是基于内存的键值数据库，它需要经常将内存中的数据同步到硬盘来保证持久化

- Redis支持两种持久化方式

  - **RDB持久化**

    - 默认的持久化方式

    - 将内存中存放的数据以快照的方式写入二进制文件，默认文件名为dump.rdb，可通过修改配置文件设置自动快照，例如：

      ```
      save 900 1                # 每900秒，数据更改1次，就发起快照保存
      save 300 10               # 每300秒，数据更改10次，则发起快照保存
      save 60  10000        	  # 每60秒，数据更改10000，则发起快照保存
      ```

    - RDB持久化产生的RDB文件是一个**经过压缩**的二进制文件，这个文件被保存在硬盘中，redis可以通过这个文件还原数据库当时的状态。 

    - 优点：**使用单独子进程来进行持久化，主进程不会进行任何IO操作，保证了redis的高性能**  

    - 缺点：**RDB是间隔一段时间进行持久化，如果持久化之间redis发生故障，会发生数据丢失。所以这种方式更适合数据要求不严谨的时候** 

    - 客户端使用save或者bgsave命令进行内存快照操作

      - save: 在主线程中保存内存快照
      - bgsave：fork一个save的子进程，在执行save过程中，不影响主进程，客户端可以正常连接redis，等子进程fork执行save完成后，通知主进程，子进程关闭 

  - **AOF持久化**

    - AOF持久化：通过保存Redis服务器锁执行的写状态来记录数据库的。 

    - 具体的讲，RDB持久化相当于备份数据库状态，而AOF持久化是备份数据库接收到的**命令**，所有被写入AOF的命令都是以redis的协议格式来保存的 

    - 服务器配置中有一项`appendfsync`，这个配置会影响服务器多久完成一次命令的记录：

      -  `always`：将缓存区的内容总是即时写到AOF文件中。
      -  `everysec`：将缓存区的内容每隔一秒写入AOF文件中。
      -  `no` ：写入AOF文件中的操作由操作系统决定，一般而言为了提高效率，操作系统会等待缓存区被填满，才会开始同步数据到磁盘。

    - redis默认采用`everysec`

    - redis在载入`AOF文件`的时候，会创建一个虚拟的client，把AOF中每一条命令都执行一遍，最终还原回数据库的状态，它的载入也是自动的。在RDB和AOF备份文件都有的情况下，redis会优先载入`AOF备份文件` 

4. ###### Redis支持主从模式
* 主从模式，即master-slave，一个master节点对应一个或多个slave

   * master负责数据存取，slave负责同步master数据，然后进行备份
   * Master 会一直将自己的数据同步更新到 Slaves 上保持主从同步
   * 只有 Master 可以执行写命令，而 Slaves 只能执行读的命令

* 数据同步过程

   * 当从库和主库建立MS关系后，会向主数据库发送SYNC命令(同步命令); 
   * 主库接收到SYNC命令后会开始在后台保存快照（RDB持久化过程），并将期间接收到的写命令缓存起来;所以就算关掉了RDB持久化方式，在他们同步的时候也会产生RDB文件 ；
   * 当快照完成后，主库会将快照文件和所有缓存的写命令发送给从库;
   * 从库接收到后，会载入快照文件并且执行收到的缓存的命令; 
   * 之后，主库每当接收到写命令时就会将命令发送从库，从而保证数据的一致。

5. ###### Redis和Memcache比较

* 性能方面两者不相上下

* 操作的便利性：redis数据结构丰富，操作方面，memcache数据结构单一

* 可靠性：redis支持数据持久化和数据恢复，而memcache不支持，通常指只作为缓存提升性能

* 数据一致性：memcacahe在并发场景下，用CAS保证一致性；redis事务支持比较弱，只能保证事务中的每个操作连续执行

* 应用场景：redis适合数据量较小的高性能操作和运算上，memcache用于在动态系统中减少数据库负载或者做缓存，提升性能； 

   

   ​    

   ​    

   ​    

   ​    

   ​    

