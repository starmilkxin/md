- [redo 日志](#redo-日志)
  - [redo 日志的作用](#redo-日志的作用)
  - [redo log的刷盘策略](#redo-log的刷盘策略)
  - [checkpoint](#checkpoint)
- [undo log](#undo-log)
  - [undo log 的作用](#undo-log-的作用)
  - [undo log 类型](#undo-log-类型)
- [binlog](#binlog)
  - [binlog 与 redo log 对比](#binlog-与-redo-log-对比)
  - [写入机制](#写入机制)
- [两阶段提交](#两阶段提交)

# redo 日志

redolog是物理日志

## redo 日志的作用

为了防止断电导致数据丢失的问题，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，由后台线程将缓存在 Buffer Pool  的脏页刷新到磁盘里，这就是  WAL （Write-Ahead Logging）技术，指的是 MySQL 的写操作并不是立刻更新到磁盘上，而是先记录在日志上，然后在合适的时间再更新到磁盘上。

- 实现事务的持久性，让 MySQL 有 crash-safe 的能力，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- 将写操作从「随机写」变成了「顺序写」，提升 MySQL 写入磁盘的性能。

![redolog流程图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/redolog流程图.png)

## redo log的刷盘策略

redo log的写入并不是直接写入磁盘的，InnoDB引擎会在写redo log的时候先写redo log buffer，之后以 一 定的频率刷入到真正的redo log file 中。

- MySQL 正常关闭时；
- 当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间的一半时，会触发落盘；
- InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。
- innodb_flush_log_at_trx_commit 参数控制。

innodb_flush_log_at_trx_commit(0、1、2，默认值为 1)

- 0；表示每次事务提交时 ，还是将 redo log 留在  redo log buffer 中 ，该模式下在事务提交时不会主动触发写入磁盘的操作。
- 1：表示每次事务提交时，都将缓存在  redo log buffer 里的  redo log 直接持久化到磁盘，这样可以保证 MySQL 异常重启之后数据不会丢失。
- 2：表示每次事务提交时，都只是缓存在  redo log buffer 里的  redo log 写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘，因为操作系统的文件系统中有个 Page Cache。

InnoDB 的后台线程每隔 1 秒：

- 针对参数 1 ：会把缓存在 redo log buffer 中的 redo log ，通过调用 write() 写到操作系统的 Page Cache，然后调用 fsync() 持久化到磁盘。所以参数为 0 的策略，MySQL 进程的崩溃会导致上一秒钟所有事务数据的丢失;
- 针对参数 2 ：调用 fsync，将缓存在操作系统中 Page Cache 里的 redo log 持久化到磁盘。所以参数为 2 的策略，较取值为 0 情况下更安全，因为 MySQL 进程的崩溃并不会丢失数据，只有在操作系统崩溃或者系统断电的情况下，上一秒钟所有事务数据才可能丢失。

![redolog刷盘](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/redolog刷盘.png)

数据安全性：参数 1 > 参数 2 > 参数 0

写入性能：参数 0 > 参数 2> 参数 1

## checkpoint

默认情况下， InnoDB 存储引擎有 1 个重做日志文件组( redo log Group），「重做日志文件组」由有 2 个 redo log 文件组成，这两个  redo  日志的文件名叫 ：ib_logfile0 和 ib_logfile1 。

InnoDB 存储引擎会先写 ib_logfile0 文件，当 ib_logfile0 文件被写满的时候，会切换至  ib_logfile1 文件，当 ib_logfile1 文件也被写满时，会切换回 ib_logfile0 文件。

![重做日志文件组写入过程](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/重做日志文件组写入过程png.png)

redo log 是循环写的方式，相当于一个环形，InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置

![重做日志文件组写入过程](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/checkpoint.png)

如果 write pos  追上了  checkpoint，就意味着 redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞（因此所以针对并发量大的系统，适当设置  redo log 的文件大小非常重要），此时会停下来将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针），然后 MySQL 恢复正常运行，继续执行新的更新操作。

所以，一次 checkpoint 的过程就是脏页刷新到磁盘中变成干净页，然后标记 redo log  哪些记录可以被覆盖的过程。

# undo log

undolog是逻辑日志

redo log是事务持久性的保证，undo log是事务原子性的保证。在事务中 更新数据 的 前置操作 其实是要 先写入一个 undo log 。

![undolog](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/undolog.png)

![undolog结构](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/undolog结构.png)

## undo log 的作用

事务需要保证原子性，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半会出现一些情况，比如:
+ 情况一:事务执行过程中可能遇到各种错误，比如服务器本身的错误，操作系统错误，甚至是突然断电导致的错误。
+ 情况二︰程序员可以在事务执行过程中手动输入ROLLBACK语句结束当前事务的执行。

以上情况出现，我们需要把数据改回原先的样子，这个过程称之为回滚，这样就可以造成一个假象∶这个事务看起来什么都没做，所以符合原子性要求。

每当我们要对一条记录做改动时(这里的改动可以指INSERT、DELETE、UPDATE)，都要留一手——把回滚时所需的东西记下来。比如:
+ 你插入一条记录时，至少要把这条记录的主键值记下来，之后回滚的时候只需要把这个主键值对应的记录删掉就好了。(对于每个INSERT, InnoDB存储引擎会完成一个DELETE)
+ 你删除了一条记录，至少要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了。(对于每个DELETE,InnoDB存储引擎会执行一个INSERT)
+ 你修改了一条记录，至少要把修改这条记录前的旧值都记录下来，这样之后回滚时再把这条记录更新为旧值就好了。(对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE,将修改前的行放回去)

MysQL把这些为了回滚而记录的这些内容称之为撤销日志或者回滚日志(即undo log)。注意，由于查询操作( SELECT）并不会修改任何用户记录，所以在查询操作执行时，并不需要记录相应的undo日志。

此外，undo log 会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。

undo log 的作用

+ 作用1︰回滚数据
用户对undo日志可能有误解:undo用于将数据库物理地恢复到执行语句或事务之前的样子。但事实并非如此。undo是逻辑日志，因此只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是数据结构和页本身在回滚之后可能大不相同。
这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在对同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。
+ 作用2: MVCC
undo的另一个作用是MVCC，即在InnoDB存储引擎中MVCC的实现是通过undo来完成。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过undo读取之前的行版本信息，以此实现非锁定读取。

对于「读提交」和「可重复读」隔离级别的事务来说，它们的快照读（普通 select 语句）是通过 Read View  + undo log 来实现的，它们的区别在于创建 Read View 的时机不同：

- 「读提交」隔离级别是在每个 select 都会生成一个新的 Read View，也意味着，事务期间的多次读取同一条数据，前后两次读的数据可能会出现不一致，因为可能这期间另外一个事务修改了该记录，并提交了事务。
- 「可重复读」隔离级别是启动事务时生成一个 Read View，然后整个事务期间都在用这个 Read View，这样就保证了在事务期间读到的数据都是事务启动前的记录。

## undo log 类型

insert undo log
- insert undo log 是指在insert操作中产生的undo log。因为insert操作的记录，只对事务本身可见，对其他事务不可见(这是事务隔离性的要求)，故该undo log可以在事务提交后直接删除。不需要进行purge操作。

update undo log
- update undo log 记录的是对delete和update操作产生的undo log。该undo log可能需要提供MVcc机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

# binlog

binlog是逻辑日志

MySQL 在完成一条更新操作后，Server 层还会生成一条 binlog，等之后事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写 入 binlog 文件。

binlog可以说是MySQL中比较重要的日志了，在日常开发及运维过程中，经常会遇到。 binlog 即 binary log，二进制日志文件，也叫作变更日志（update log）。它记录了数据库所有执行的 DDL 和 DML 等数据库更新事件的语句，但是不包含没有修改任何数据的语句（如数据查询语句select、 show等）。它以事件形式记录并保存在二进制文件中。通过这些信息，我们可以再现数据更新操作的全过程。如果想要记录所有语句（例如，为了识别有问题的查询)，需要使用通用查询日志。

binlog 主要应用场景:

- 一是用于数据恢复，如果MySQL数据库意外停止，可以通过二进制日志文件来查看用户执行了哪些操作，对数据库服务器文件做了哪些修改，然后根据二进制日志文件中的记录来恢复数据库服务器。
- 二是用于数据复制，由于日志的延续性和时效性，master把它的二进制日志传递给slaves来达到master-slave数据一致的目的。

可以说MySQL数据库的数据备份、主备、主主、主从都离不开binlog，需要依靠binlog来同步数据，保证数据一致性。

## binlog 与 redo log 对比

适用对象不同:

- binlog 是 MySQL 的 Server 层实现的日志，所有存储引擎都可以使用，记录内容是语句的原始逻辑，类似于“给ID=2这一行的c字段加1”，是逻辑日志；
- redo log 是 Innodb 存储引擎实现的日志，记录内容是“在某个数据页上做了什么修改”，是物理日志；

用途不同:

- redo log让 InnoDB 存储引擎拥有了崩溃恢复能力。
- binlog 保证了 MySQL 集群架构的数据一致性。

写入方式不同:

- binlog 是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖以前的日志，保存的是全量的日志。
- redo log 是循环写，日志空间大小是固定，全部写满就从头开始，保存未被刷入磁盘的脏页日志。

## 写入机制

binlog 的写入时机也非常简单，事务执行过程中，先把日志写到 binlog cache ，事务提交的时候，再 把 binlog cache 写到binlog文件中。因为一个事务的 binlog 不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为 binlog cache。
我们可以通过 binlog_cache_size 参数控制单个线程 binlog cache 大小，如果存储内容超过了这个参数，就要暂存到磁盘(Swap) 。binlog 日志刷盘流程如下:

![binlog1](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/binlog1.png)

上图的write，是指把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，所以速度比较快。上图的fsync，才是将数据持久化到磁盘的操作。

write和fisync的时机，可以由参数sync_binlog控制，默认是日。为0的时候，表示每次提交事务都只write，由系统自行判断什么时候执行fsync。虽然性能得到提升，但是机器宕机, page cache里面的 binglog 会丢失。如下图:

![binlog2](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/binlog2.png)

为了安全起见，可以设置为 1 ，表示每次提交事务都会执行fsync，就如同redo log 刷盘流程一样。 最后还有一种折中方式，可以设置为N(N>1)，表示每次提交事务都write，但累积N个事务后才fsync。

![binlog3](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/binlog3.png)

在出现IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能。同样的，如果机器宕
机，会丢失最近N个事务的binlog日志。

# 两阶段提交

在持久化 redo log 和 binlog 这两份日志的时候，如果出现半成功的状态，就会造成主从环境的数据不一致性。这是因为 redo log 影响主库的数据，binlog 影响从库的数据，所以 redo log 和 binlog 必须保持一致才能保证主从数据一致。

在执行更新语句过程，会记录 redo log 与 binlog 两块日志，以基本的事务为单位，redo log 在事务执行过程 中可以不断写入，而 binlog 只有在提交事务时才写入，所以 redo log 与 bin log 的 写入时机 不一样。

为了解决两份日志之间的逻辑一致问题，InnoDB 存储引擎使用两阶段提交方案。原理很简单，将 redo log 的写入拆成了两个步骤 prepare 和 commit，这就是两阶段提交。

![两阶段1](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/两阶段1.png)

使用两阶段提交后，写入 binlog 时发生异常也不会有影响，因为 MySQL 根据 redo log 日志恢复数据时，发现 redolog 还处于 prepare 阶段，并且没有对应 binlog 日志，就会回滚该事务。

![两阶段2](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/两阶段2.png)

另一个场景，redo log 设置 commit 阶段发生异常，同样也不会回滚事务，它会执行上图框住的逻辑，虽然 redo log 是处于 prepare 阶段，但是能通过事务 id 找到对应的 binlog 日志，所以 MySQL 认为是完整的，就会提交事务恢复数据。
