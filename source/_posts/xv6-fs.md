---
title: xv6-fs
urlname: RLSwdlgCRo68xTxoNn6caCU1nRh
date: 2023-10-17T21:09:23.000Z
updated: '2024-01-18 17:32:05'
tags:
  - 公开课
  - MIT6.s081
  - File System
  - xv6
categories: 笔记
---
> 下文中没有具体说明的场景均描述 xv6 文件系统的实现



好吧 我们就来详细捋一捋 fs


## **组成**


// todo


## **事务 Transaction**


> 事务在用数据库的时候就已经经常用到了，但是没想到文件系统也有事务。从文件系统的事务实现中我们也可以管中窥豹数据库的事务是怎么实现的，数据库的 crash recovery 是如何保证的。


### **xv6**


事务的作用是保证一组磁盘写操作的原子性 （即要么全部完成，要么全部失败)，从而支持 crash recovery。事务是我们对一组整体具有原子性的磁盘写操作的抽象，而我们是如何实现事务这个特性的呢？



当然是凭借日志系统。在xv6中，我们在执行一组磁盘写操作时，会先将这组写操作发到 block cache。block cache 就是磁盘 block 在内存中的 copy。



在write系统调用的最后（也就是事务结束时），这些更新都被从 block cache 拷贝到了 log 中，之后会更新 header block 的计数来表明当前的 transaction 已经结束了。



更新 header block 的计数这个操作是一个原子操作（得益于硬件支持），被称为 commit point。如果系统在 commit point 之前崩溃，则相当于事务在执行时中断，我们便放弃之前已有的全部更改。但如果系统在 commit point 之后崩溃，则相当于事务已经执行完毕（但是还没有回收）。虽然此时 log 中的变更还没有写入到磁盘对应的位置，但是即使在此时崩溃重启时也可以根据 log 中的记录恢复之前的更改。



commit point 之后当然是将 log 中的变更写回磁盘，然后清除 header block 中的计数（计数清除被视作释放了一个事务），至此一个事务的流程完成。


### **ext3**


ext3 是在真实世界中使用的一种文件系统实现，直到现在也仍然经常在 linux 上使用。我们可以轻易的在 linux 上以 ext3 文件系统的格式挂载一块硬盘：


```bash
sudo vim /etc/fstab # 在文件末尾加上 /dev/<设备> <要挂载的路径> ext3 defaults 1 1
```


ext3 的实现思路是基于 [Journaling the Linux ext2fs Filesystem](https://pdos.csail.mit.edu/6.828/2020/readings/journal-design.pdf) 中描述的一种高性能高可靠文件系统实现。其实现方式跟 xv6 的文件系统实现有很多类似的地方，不过因为其用于现实世界，所以比 xv6 考虑了更多细节，且总体上性能要更好（好得多）：


- ext3 提供了异步系统调用 ，即系统修改完 block cache 就直接返回（是的，ext3 上的 write 系统调用返回时数据其实还没有被写到磁盘上，在这个中途 crash 事务是有可能回滚的）

- ext3 支持批处理 (batching) ，即 ext3 可以合并一段时间内的所有事务，最后将其作为一个大事务进行提交，这样做整体只会向磁盘中写入一次数据。
	- 首先批处理减少了事务带来的开销，从机械硬盘中查找log的位置实际上开销不小，而批处理合并了多个事务，减少了事务执行的次数。
		- 如果之前的多个事务中都修改了同一个block，批处理后写入磁盘时就只需要写入一次 (这种情况被称作 write absorbing)，相当于减少了磁盘 io。
		- 批处理利好 disk scheduling
		- 磁盘的读写也是具有局部性的，一次性的向磁盘的连续位置写入1000个block，要比分1000次每次写一个不同位置的磁盘block快得多。写log就是向磁盘的连续位置写block。通过向磁盘提交大批量的写操作，可以更加的高效。
			- 如果能将大量的写请求同时发送到驱动，即使它们位于磁盘的不同位置，我们也使得磁盘可以调度这些写请求，并以特定的顺序执行这些写请求，这也很有效。在一个机械硬盘上，如果一次发送大量需要更新block的写请求，驱动可以对这些写请求根据轨道号排序。甚至在一个固态硬盘中，通过一次发送给硬盘大量的更新操作也可以稍微提升性能。所以，只有发送给驱动大量的写操作，才有可能获得disk scheduling。这是batching带来的另一个好处。
		
- 良好的并发性能 concurrency
	- ext3 允许并行写操作，在一个总的事务提交之前，其他 write 系统调用不必等待到当前事务执行完毕，而是直接让 write 操作合并到当前事务当中。
		- 可以有不同的 transaction 同时存在，尽管只有一个 open transaction 可以接收系统调用。
		- 一个 open transaction
			- 若干个正在commit到log的transaction，我们并不需要等待这些transaction结束。当之前的transaction还没有commit并还在写log的过程中，新的系统调用仍然可以在当前的open transaction中进行。
			- 若干个正在从cache中向文件系统block写数据的transaction
			- 若干个正在被释放的transaction，这个并不占用太多的工作
			- 也就是说 提交事务 ，写入磁盘，释放事务 这三个操作都是异步的。
		



