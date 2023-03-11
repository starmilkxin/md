InnoDB中为了能够高效地，不冲突地进行数据的读写，所以采用了MVCC+临建锁的方法。<br/>
我们主要讨论MVCC是如何解决幻读问题的。首先是幻读的通俗理解就是第一次SELECT为N条数据，而第二次SELECT时得到的数据行数大于N，仿佛出现了幻觉一样，所以叫幻读。<br/>
InnoDB开启RR隔离级别后，事务的第一次SELECT时，会产生一个ReadView，**注意，RC隔离级别与RR隔离级别的区别就在于，RC是事务每次SELECT都会产生最新的ReadView，后RR事务中第一次SELECT中产生一个新的ReadView，此后这个事务的SELECT都采用同样的ReadView**。<br/>
ReadView简图
![ReadView](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220217190114.png)

不加锁的SELECT为快照读，根据ReadView中的事务ID，来判断是否能读，如果不能，那就使用undo log来查找其历史版本，直到符合规则或是返回null。 而如果是加锁的SELECT, UPDATE, DELETE的话，那么就会使用当前读，会更新新的ReadView，并将更新操作写入undo log放入链表中。
![undo log](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220217185934.png)
<br/>
令我困惑的就是，我们知道，快照读和当前读结合使用的话，依旧是会出现幻读的情况的。当我自己试验时，确实INSERT的情况是会出现的，但是DELETE的情况并没有出现，这就很令我费解。

|事务A|事务B|
|---|---|
|A开启 | |
|A SELECT ，得到5条数据|B开启  |
| |B SELECT ，得到5条数据 |
| |B DELETE 最后一条数据|
| |B COMMIT |
|A UPDATE 所有数据 | |
|A SELECT， 得到5条数据 | |

我一开始是认为应该在UPDATE后，再SELECT是得到4条数据的。之后，个人理解A在UPDATE时，发现了4条数据，于是对他们加了临建锁，修改值后，将原值加入undo log中，所以第5条数据在ReadView中是没有被改动的，因此当A再次 SELECT时，还是可以得到第5条数据。<br/>
<br/>
令我困惑的第二个问题就是，为什么undo log要分类为insert undo log和update undo log（update undo log包括update和delete）。大众的解释就是说insert undo log在事务commit后，可以直接删除，而update undo log却不行，因为MVCC需要其得到历史数据，那为什么MVCC不需要insert undo log来得到历史数据呢？<br/>
后来转念一想，噢~原来是因为，如果该数据的上一个版本的历史数据的undo log是insert类型的话，那么就代表上一版本压根就没有这个数据。只有上个版本没有这个数据，才会insert，然后产生这个版本的数据。因此我们只需要知道数据的undo log类型是insert，那么其历史数据就是null，而不需要再浪费空间存储undo insert log了。再者而言，undo log的另一作用，回滚数据，在commit后已经数据已经持久化了，综上而言，insert undo log在commit就没有任何存在价值了。