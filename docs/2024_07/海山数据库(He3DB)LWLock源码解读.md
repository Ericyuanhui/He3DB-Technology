# 海山数据库(He3DB)源码解读：海山PG LWLock实现

# 背景

He3DB for PostgreSQL是受Aurora论文启发，基于开源数据库PostgreSQL 改造的数据库产品。架构上实现计算存储分离，并进一步支持数据的冷热分层，大幅提升产品的性价比。
He3DB for PostgreSQL中存在多个会话试图同时访问同一数据的情况，并发控制的目标就是保证所有会话高效地访问，同时维护数据完整性，并发访问控制的常用方式为两种：锁机制和多版本并发控制（MVCC）。因为 MVCC 并不能解决所有的并发控制情况，所以还需要使用传统的锁机制来保证那些通常不需要完整事务隔离并且想要显式管理特定冲突点的应用。

# 整体概述

按照功能划分，锁管理分为锁功能模块，锁级别管理模块，死锁处理模块。
锁功能模块：针对三种类型的锁功能，自旋锁，轻量级锁，事务锁。
锁级别管理模块：针对四种不同级别的锁管理器，表级别、页级别、元组级别、事务级别。
死锁处理模块：包括死锁检测功能和死锁处理功能。


![图1 锁管理模块介绍 ](https://github.com/Ericyuanhui/He3DB_Technology/blob/main/docs/images/%E9%94%81%E5%8A%9F%E8%83%BD%E6%A8%A1%E5%9D%97%E5%88%92%E5%88%86.png)

#  数据结构

LWLock(轻量级锁)主要提供对共享的数据结构的互斥访问。LWLock有两种锁模式，一种为排他模式，另一种为共享模式。
轻量级锁的特点是：LWLock的特点是:有等待队列、无死锁检测、能自动释放锁。
核心数据结构如下：


 - LWLockMode 锁模式
```javascript
 typedef enum LWLockMode
{
	LW_EXCLUSIVE,
	LW_SHARED,
	LW_WAIT_UNTIL_FREE			/* A special mode used in PGPROC->lwWaitMode,
								 * when waiting for lock to become free. Not
								 * to be used as LWLockAcquire argument */
} LWLockMode;
```

- LWLock 轻量锁定义
```javascript
typedef struct LWLock
{
	uint16		tranche;		/* tranche ID */
	pg_atomic_uint32 state;		/* state of exclusive/nonexclusive lockers */
	proclist_head waiters;		/* list of waiting PGPROCs */
#ifdef LOCK_DEBUG
	pg_atomic_uint32 nwaiters;	/* number of waiters */
	struct PGPROC *owner;		/* last exclusive owner of the lock */
#endif
} LWLock;
```
- lwlock_stats 用来记录lwlock的状态结构体，使用一个hash_table来记录所有的
```javascript
typedef struct lwlock_stats
{
	lwlock_stats_key key;
	int			sh_acquire_count;
	int			ex_acquire_count;
	int			block_count;
	int			dequeue_self_count;
	int			spin_delay_count;
}lwlock_stats;
```
- LWLockHandle  定义了LWLockHandle held_lwlocks[MAX_SIMUL_LWLOCKS]，用来保存全局的申请成功的LWLock
```javascript
typedef struct LWLockHandle
{
	LWLock	   *lock;
	LWLockMode	mode;
} LWLockHandle;
```
- LWLockWaitState 进程process的state of the wait process
```javascript
typedef enum LWLockWaitState
{
	LW_WS_NOT_WAITING, /* not currently waiting / woken up */
	LW_WS_WAITING, /* currently waiting */
	LW_WS_PENDING_WAKEUP /* removed from waitlist, but not yet signalled */
} LWLockWaitState;
```
系统使用一个全局的数组MainLWLockArray来管理所有的LWLock，这些LWLock定义lwlocknames.txt中，当前定义了47种LWLock来实现分段锁表，从而实现用户自定义的锁类型。
```javascript
ShmemIndexLock						1
OidGenLock							2
XidGenLock							3
ProcArrayLock						4
SInvalReadLock						5
SInvalWriteLock						6
WALBufMappingLock					7
WALWriteLock						8
ControlFileLock						9
# 10 was CheckpointLock
XactSLRULock						11
SubtransSLRULock					12
MultiXactGenLock					13
MultiXactOffsetSLRULock				14
MultiXactMemberSLRULock				15
RelCacheInitLock					16
CheckpointerCommLock				17
TwoPhaseStateLock					18
TablespaceCreateLock				19
BtreeVacuumLock						20
AddinShmemInitLock					21
AutovacuumLock						22
AutovacuumScheduleLock				23
SyncScanLock						24
RelationMappingLock					25
NotifySLRULock						26
NotifyQueueLock						27
SerializableXactHashLock			28
SerializableFinishedListLock		29
SerializablePredicateListLock		30
SerialSLRULock						31
SyncRepLock							32
BackgroundWorkerLock				33
DynamicSharedMemoryControlLock		34
AutoFileLock						35
ReplicationSlotAllocationLock		36
ReplicationSlotControlLock			37
CommitTsSLRULock					38
CommitTsLock						39
ReplicationOriginLock				40
MultiXactTruncationLock				41
OldSnapshotTimeMapLock				42
LogicalRepWorkerLock				43
XactTruncationLock					44
# 45 was XactTruncationLock until removal of BackendRandomLock
WrapLimitsVacuumLock				46
NotifyQueueTailLock					47
```


# LWLock设计
## 设计原理

LWLock(轻量级锁)主要提供对共享的数据结构的互斥访问。LWLock有两种锁模式，一种为排他模式，另一种为共享模式。轻量级锁不提供死锁检测，但轻量级锁管理器在恢复期间被自动释放，所以持有轻量级锁的期间调用恢复功能发生错误，不会出现轻量级锁未释放的问题。LWLock利用SpinLock实现，当没有锁的竞争时可以很快获得或释放LWLock。当一个进程阻塞在一个轻量级锁上时，相当于它阻塞在一个信号量上，所以不会消耗CPU时间，等待的进程将会以先来后到的顺序被授予锁。


##  主要流程
LWlock的主要操作流程是：

1.LWLock的空间分配： Postmaster启动后需要在共享内存空间中为LWLock分配空间，分配空间时，需要计算出将要分配的LWLock的个数。

2.LWlock的创建 ：LWLock的创建过程包括分配空间和初始化两个过程。其中分配空间的过程在LWLockShmemSize函数中给出，初始化即对LWLock数据库结构的初始化，使它们处于“未上锁”的状态，并在LWLockArray队列的末尾初始化动态分配计数器。

3.LWLock的分配：LWLock 分配操作是从共享内存中预定义的闲置LWLock Aray中得到一个LWLock，同时LWlockAmay前面的计数器会累加。基本过程是申请锁，使用计数器得到闲置的LWLock 数目，若没有闲置的LWlock，输出错误;否则，修改计数器，释放锁，返回指向闲置LWLock 的一个指针。

4.LWLock锁的获取：LWLock的获取由函数LWLockAcquire定义，该函数试图以给定的轻量级锁ID(LWLock ID)和锁模式(SHARE/EXCLUSIVE)获得对应的轻量级锁。如果暂时不能获得，就进入睡眠状态直至该锁空闲。该函数执行流程如下:

 - 开中断，获取锁。
 - 检查锁的当前情况，如果空闲则获得锁，然后退出:
 - 否则把自己加入等待队列中，并一直等到被唤醒。
 - 重复上述步骤。
LWLock还提供了另外一种获取LWLock的方式---使用LWLockConditionalAcquire 函数，它与上述函数的区别是若能获得此锁，则返回TRUE，否者返回FALSE。

5.LWLock锁的释放：LWIock的释放由函数LWLockRelease完成，主要功能是释放指定LockID的锁。该函数的执行流程如下:

 - 获得该锁。
 - 检查此次解锁是否唤醒其他等待进程。
 - 如果需要唤醒进程，遍历等待队列;如果遇到要求读锁的进程，从队列中删除，但保留一个指向它的指针。重复操作直到遇到要求写锁的进程。
 - 释放该LWLock，把从队列中删除的进程唤醒。
另外提供能释放当前后端持有的所有LWLock锁的功能，该功能在函数LWLockReleaseAll中定义，主要在系统出现错误之后使用。
![](https://github.com/Ericyuanhui/He3DB_Technology/blob/main/docs/images/LWLock%E4%BD%BF%E7%94%A8%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 主要接口
| 对外接口函数  | 接口说明  |
| ------------ | ------------ |
|  void CreateLWLocks(void) |  创建LWLock |
| Size LWLockShmemSize(void)  | LWLock空间分配  |
|  bool LWLockAcquire(LWLock *lock, LWLockMode mode)| 申请LWLock  |
| bool LWLockConditionalAcquire(LWLock *lock, LWLockMode mode)  | 有条件申请LWLock  |
| void LWLockRelease(LWLock *lock)  | 释放LWLock  |
| void LWLockReleaseAll(void) |  释放全部锁 |

## 源码流程图
![在这里插入图片描述](https://github.com/Ericyuanhui/He3DB_Technology/blob/main/docs/images/He3DB%E9%94%81%E5%AE%9E%E7%8E%B0%E8%AE%BE%E8%AE%A1LWlock.png)

# 作者介绍
徐元慧，移动云数据库系统架构师，负责云原生数据库He3DB的架构设计与研发。
