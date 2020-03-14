---
title: Redis那么快的背后
top: true
cover: false
toc: true
mathjax: true
date: 2020-02-16 15:09:23
password:
summary: 
tags:
- Java
categories:
- Java
---

## 前言

Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（Transactions） 和不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）。

<!--more-->


## 一、I/O 多路复用（ I/O Multiplexing）

### 1.1、传统 I/O 数据拷贝

当应用程序执行 read 系统调用读取文件描述符（FD）的时候，如果这块数据已经存在于用户进程的页内存中，就直接从内存中读取数据。如果数据不存在，则先将数据从磁盘加载数据到内核缓冲区中，再从内核缓冲区拷贝到用户进程的页内存中。（两次拷贝，两次 user 和 kernel 的上下文切换）。

为了解决阻塞的问题，我们有几个思路。
1、在服务端创建多个线程或者使用线程池，但是在高并发的情况下需要的线程会很多，系统无法承受，而且创建和释放线程都需要消耗资源。
2、由请求方定期轮询，在数据准备完毕后再从内核缓存缓冲区复制数据到用户空间（非阻塞式 I/O），这种方式会存在一定的延迟。

## 1.2、redis是怎么使用单线程去处理多个客户端请求？

答案：I/O 多路复用（ I/O Multiplexing）
I/O 指的是网络 I/O。
多路指的是多个 TCP 连接（Socket 或 Channel）。
复用指的是复用一个或多个线程。
它的基本原理就是不再由应用程序自己监视连接，而是由内核替应用程序监视文件描述符。

 客户端在操作的时候，会产生具有不同事件类型的 socket。在服务端，I/O 多路复
用程序（I/O Multiplexing Module）会把消息放入队列中，然后通过文件事件分派器（File
event Dispatcher），转发到不同的事件处理器中。  

![avatar](https://s2.ax1x.com/2020/02/26/3NfIOg.png)

  多路复用有很多的实现，以 select 为例，当用户进程调用了多路复用器，进程会被阻塞。内核会监视多路复用器负责的所有 socket，当任何一个 socket 的数据准备好了，多路复用器就会返回。这时候用户进程再调用 read 操作，把数据从内核缓冲区拷贝到用户空间  

select函数原型

```c++
#include <sys/select.h>
/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,
			fd_set *exceptfds, struct timeval *timeout);

	nfds: 		监控的文件描述符集里最大文件描述符加1，因为此参数会告诉内核检测前多少个文件描述符的状态
	readfds：	监控有读数据到达文件描述符集合，传入传出参数
	writefds：	监控写数据到达文件描述符集合，传入传出参数
	exceptfds：	监控异常发生达文件描述符集合,如带外数据到达异常，传入传出参数
	timeout：	定时阻塞监控时间，3种情况
				1.NULL，永远等下去
				2.设置timeval，等待固定时间
				3.设置timeval里时间均为0，检查描述字后立即返回，轮询
	struct timeval {
		long tv_sec; /* seconds */
		long tv_usec; /* microseconds */
	};
	void FD_CLR(int fd, fd_set *set); 	//把文件描述符集合里fd清0
	int FD_ISSET(int fd, fd_set *set); 	//测试文件描述符集合里fd是否置1
	void FD_SET(int fd, fd_set *set); 	//把文件描述符集合里fd位置1
	void FD_ZERO(fd_set *set); 			//把文件描述符集合里所有位清0
```

```c++
/* server.c */
#define MAXLINE 80
#define SERV_PORT 6666

int main(int argc, char *argv[])
{
	int i, maxi, maxfd, listenfd, connfd, sockfd;
	int nready, client[FD_SETSIZE]; 	/* FD_SETSIZE 默认为 1024 */
	ssize_t n;
	fd_set rset, allset;
	char buf[MAXLINE];
	char str[INET_ADDRSTRLEN]; 			/* #define INET_ADDRSTRLEN 16 */
	socklen_t cliaddr_len;
	struct sockaddr_in cliaddr, servaddr;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

bzero(&servaddr, sizeof(servaddr));
servaddr.sin_family = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
servaddr.sin_port = htons(SERV_PORT);

Bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

Listen(listenfd, 20); 		/* 默认最大128 */

maxfd = listenfd; 			/* 初始化 */
maxi = -1;					/* client[]的下标 */

for (i = 0; i < FD_SETSIZE; i++)
	client[i] = -1; 		/* 用-1初始化client[] */

FD_ZERO(&allset);
FD_SET(listenfd, &allset); /* 构造select监控文件描述符集 */

for ( ; ; ) {
	rset = allset; 			/* 每次循环时都从新设置select监控信号集 */
	nready = select(maxfd+1, &rset, NULL, NULL, NULL);

	if (nready < 0)
		perr_exit("select error");
	if (FD_ISSET(listenfd, &rset)) { /* new client connection */
		cliaddr_len = sizeof(cliaddr);
		connfd = Accept(listenfd, (struct sockaddr *)&cliaddr, &cliaddr_len);
		printf("received from %s at PORT %d\n",
				inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
				ntohs(cliaddr.sin_port));
		for (i = 0; i < FD_SETSIZE; i++) {
			if (client[i] < 0) {
				client[i] = connfd; /* 保存accept返回的文件描述符到client[]里 */
				break;
			}
		}
		/* 达到select能监控的文件个数上限 1024 */
		if (i == FD_SETSIZE) {
			fputs("too many clients\n", stderr);
			exit(1);
		}

		FD_SET(connfd, &allset); 	/* 添加一个新的文件描述符到监控信号集里 */
		if (connfd > maxfd)
			maxfd = connfd; 		/* select第一个参数需要 */
		if (i > maxi)
			maxi = i; 				/* 更新client[]最大下标值 */

		if (--nready == 0)
			continue; 				/* 如果没有更多的就绪文件描述符继续回到上面select阻塞监听,
										负责处理未处理完的就绪文件描述符 */
		}
		for (i = 0; i <= maxi; i++) { 	/* 检测哪个clients 有数据就绪 */
			if ( (sockfd = client[i]) < 0)
				continue;
			if (FD_ISSET(sockfd, &rset)) {
				if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
					Close(sockfd);		/* 当client关闭链接时，服务器端也关闭对应链接 */
					FD_CLR(sockfd, &allset); /* 解除select监控此文件描述符 */
					client[i] = -1;
				} else {
					int j;
					for (j = 0; j < n; j++)
						buf[j] = toupper(buf[j]);
					Write(sockfd, buf, n);
				}
				if (--nready == 0)
					break;
			}
		}
	}
	close(listenfd);
	return 0;
}
```



## 二、 内存回收

### 2.1、内存回收—— LRU 淘汰原理

#### 问题1：基于一个数据结构做缓存，怎么实现 LRU——最长时间不被访问的元素在超过容量时删除？
常规的哈希表+双向链表
但存在的问题在于，基于传统的LRU算法实现Redis LRU需要额外的数据结构存储，消耗内存

Redis LRU 对传统的 LRU 算法进行了改良，通过随机采样来调整算法的精度。
如果淘汰策略是 LRU，则根据配置的采样值 maxmemory_samples（默认是 5 个）,
随机从数据库中选择 m 个 key, 淘汰其中热度最低的 key 对应的缓存数据。所以采样参数m配置的数值越大, 就越能精确的查找到待淘汰的缓存数据,但是也消耗更多的CPU计算,执行效率降低。
而听说 Redis LRU 算法在 sample 为 10 的情况下，已经能接近传统 LRU 算法了

#### 问题2：如何找出热度最低的数据？
Redis 中所有对象结构都有一个 lru 字段, 且使用了 unsigned 的低 24 位，这个字段用来记录对象的热度。对象被创建时会记录 lru 值。在被访问的时候也会更新 lru 的值。

源码：server.c

```c++
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
    * LFU data (least significant 8 bits frequency
    * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

但是对于访问时间的计算不是获取系统当前的时间戳，而是设置为全局变量 。Redis 中 有 个 定 时 处 理 的 函 数 serverCron ， 默 认 每 100 毫 秒 调 用 函 数updateCachedTime 更新一次全局变量的 server.lruclock 的值，它记录的是当前 unix时间戳。这样函数 lookupKey 中更新数据的 lru 热度值时,就不用每次调用系统函数 time，可以提高执行效率  Redis 中 有 个 定 时 处 理 的 函 数 serverCron ， 默 认 每 100 毫 秒 调 用 函 数updateCachedTime 更新一次全局变量的 server.lruclock 的值，它记录的是当前 unix时间戳。这样函数 lookupKey 中更新数据的 lru 热度值时,就不用每次调用系统函数 time，可以提高执行效率  

函数 estimateObjectIdleTime 评估指定对象的 lru 热度，思想就是对象的 lru 值和全局的 server.lruclock 的差值越大（越久没有得到更新）， 该对象热度越低。  

```c++
/* Given an object returns the min number of milliseconds the object was never
* requested, using an approximated LRU algorithm. */
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
    	return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
    	return (lruclock + (LRU_CLOCK_MAX - o->lru)) *LRU_CLOCK_RESOLUTION;
	}
}
```

### 2.1、内存回收—— LFU 淘汰原理

```c++
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
    * LFU data (least significant 8 bits frequency
    * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

  当这 24 bits 用作 LFU 时，其被分为两部分：高 16 位用来记录访问时间（低 8 位用来记录访问频率，简称 counter）。counter 是用基于概率的对数计数器实现的，8 位可以表示百万次的访问频率，对象被读写的时候，lfu 的值会被更新。  

## 三、Redis的持久化方案 

### 3.1 RDB 

RDB 是一个非常紧凑(compact)的文件，它保存了 redis 在某个时间点上的数据集。这种文件非常适合用于进行备份和灾难恢复。

这里就提一下bgsave，执行 bgsave 时，Redis 会在后台异步进行快照操作的同时还可以响应客户端请求。
具体操作是 Redis 进程执行 fork 操作创建子进程（copy-on-write），RDB 持久化过程由子进程负责，完成后自动结束。它不会记录 fork 之后后续的命令。阻塞只发生在fork 阶段，一般时间很短。

如果数据相对来说比较重要，希望将损失降到最小，则可以使用 AOF 方式进行持久

可以手动调用 lastsave 命令查看最近一次成功生成快照的时间  

### 3.2、AOF

 AOF 持久化是 Redis 不断将写命令记录到 AOF 文件中，随着 Redis 不断的进行，AOF 的文件会越来越大，文件越大，占用服务器内存越大以及 AOF 恢复要求时间越长。
为了解决这个问题，Redis 新增了重写机制，当 AOF 文件的大小超过所设定的阈值时，Redis 就会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集。AOF 文件重写并不是对原文件进行重新整理，而是直接读取服务器现有的键值对，然后用一条命令去代替之前记录这个键值对的多条命令，生成一个新的文件后去替换原来的 AOF 文件 

可以使用命令 bgrewriteaof 来重写。

## 四、Redis集群高可用下的数据分片

  数据分片有几个关键的问题需要解决：
1、数据怎么相对均匀地分片
2、客户端怎么访问到相应的节点和数据
3、重新分片的过程，怎么保证正常服务  

针对Redis集群高可用下的数据分片，Redis 既没有用哈希取模，也没有用一致性哈希，而是用虚拟槽来实现的。Redis 创建了 16384 个槽（slot），每个节点负责一定区间的 slot。Redis 的每个 master 节点维护一个 16384 位（2048bytes=2KB）的位序列，比如：序列的第 0 位是 1，就代表第一个 slot 是它负责；序列的第 1 位是 0，代表第二个 slot不归它负责。当我们敲下set xx xx时候，Redis会通过对 key 用 CRC16 算法计算再%16384得到一个 slot的值，数据落到负责这个 slot 的 Redis 节点上。于是乎key 与 slot 的关系是永远不会变的，会变的只有 slot 和 Redis 节点的关系  

当然我们总有这样的需求，希望不同的key落到相同的Redis节点上，以减少网络io的次数，那么我们就可以通过

```
set a{qs}a 1
```

Redis 在计算槽编号的时候只会获取{}之间的字符串进行槽编号计算，  在 key 里面加入相同的{hash tag}保证上面两个不同的键被计算进入相同的槽中，这样就会落入redis集群中同一个节点中



 ##  五、Redis数据与数据库数据的一致性问题

**矛盾集中在：到底是先更新数据库，再删除缓存，还是先删除缓存，再更新数据库  **

### 5.1 先更新数据库， 再删除缓存

正常情况：
更新数据库，成功。
删除缓存，成功。

异常情况：
1、更新数据库失败，程序捕获异常，不会走到下一步，所以数据不会出现不一致。
2、更新数据库成功，删除缓存失败。数据库是新数据，缓存是旧数据，发生了不一致的情况。

**异步更新缓存：**

因为更新数据库时会往 binlog 写入日志，所以我们可以通过一个服务来监听 binlog的变化（比如阿里的canal），然后在客户端完成删除 key 的操作。如果删除失败的话，再发送到消息队列。总之，对于后删除缓存失败的情况，我们的做法是不断地重试删除，直到成功。（摘自网络）

###  5.2 先删除缓存， 再更新数据库

正常情况：
删除缓存，成功。
更新数据库，成功。
异常情况：
1、删除缓存，程序捕获异常，不会走到下一步，所以数据不会出现不一致。
2、删除缓存成功，更新数据库失败。 因为以数据库的数据为准，所以不存在数据不一致的情况。

**看起来好像没问题，但是如果有程序并发操作的情况下：
1）线程 A 需要更新数据，首先删除了 Redis 缓存
2）线程 B 查询数据，发现缓存不存在，到数据库查询旧值，写入 Redis，返回
3）线程 A 更新了数据库
这个时候，Redis 是旧的值，数据库是新的值，发生了数据不一致的情况。
那问题就变成了：能不能让对同一条数据的访问串行化呢？代码肯定保证不了，因为有多个线程，即使做了任务队列也可能有多个服务实例。数据库也保证不了，因为会有多个数据库的连接。只有一个数据库只提供一个连接的情况下，才能保证读写的操作是串行的，或者我们把所有的读写请求放到同一个内存队列当中，但是这种情况吞吐量太低了。**
所以我们有一种延时双删的策略，在写入数据之后，再删除一次缓存。

A 线程：
1）删除缓存
2）更新数据库
3）休眠 500ms（这个时间，依据读取数据的耗时而定）
4）再次删除缓存
