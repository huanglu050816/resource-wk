## mysql锁概述  
* 相对于其他数据库的锁，mysql的锁机制比较简单；不同的存储引擎，锁机制不同。  
* MyISAM和MEMORY存储引擎采用的是表级锁（table-level locking）。  
* BerkeleyDB(BDB) 存储引擎采用的是页面锁（page-level locking）,也支持表级锁。  
* InnoDB存储引擎既支持行级锁（row-level locking），也支持表级锁。  
  
* 表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低。  
* 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。  
* 页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般  
  
## MyISAM
### MyISAM表锁
* mysql的表锁有两种模式：1、表共享读锁（Table Read Lock）,表独占锁（Table Write Lock）
* 当一个session获得一个表的写锁后，只有获得写锁的session可以对表进行读写操作，其他session的读、写都会等待，直到锁被释放为止。  
* 当一个session获得一个表的读锁后，这个session可以查询锁定表中的记录，但是当前session不能查询没有锁定的表，当前session插入和更新锁定表也会报错；其他session可以查询锁定表中的记录，其他session也可以查询和更新其他表未锁定的表，其他session更新锁定表会等待。

### MyISAM如何加锁
* select 给涉及的所有表加读锁  
* UPDATE、DELETE、INSERT 自动给涉及的表加写锁  
* 显示加锁 LOCK TABLES，但是在显示加锁之后 只能访问加锁了的表，不能访问其他表了。所以显示加锁，只能一次性把所需要的表都锁上，如果存在别名，也要加上别名一起锁上。比如:lock table actor as a read,actor as b read;
### MyISAM并发插入（Concurrent Inserts）
MyISAM存储引擎有一个系统变量concurrent_insert，专门用以控制其并发插入的行为，其值分别可以为0、1或2。
* 当concurrent_insert设置为0时，不允许并发插入。
* 当concurrent_insert设置为1时，如果MyISAM表中没有空洞（即表的中间没有被删除的行），MyISAM允许在一个进程读表的同时，另一个进程从表尾插入记录。这也是MySQL的默认设置。
* 当concurrent_insert设置为2时，无论MyISAM表中有没有空洞，都允许在表尾并发插入记录。

## InnoDB
### InnoDB行锁  
InnoDB实现了以下两种类型的行锁。
* 共享锁（s）：又称读锁。若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。
* 排他锁（x）：又称写锁。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。

### InnoDB如何加行锁
* select语句默认不会加任何锁类型
* update,delete,insert都会自动给涉及到的数据加上排他锁
* select …for update 加排他锁
* 显性加共享锁 select … lock in share mode  

所以加过排他锁的数据行在其他事务种是不能修改数据的，也不能通过for update和lock in share mode锁的方式查询数据，但可以直接通过select …from…查询数据，因为普通查询没有任何锁机制。

### InnoDB行锁实现方式
* InnoDB行锁是通过给索引上的索引项加锁来实现的，这一点MySQL与Oracle不同，后者是通过在数据块中对相应数据行加锁来实现的。
* 只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！
* 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。
* 使用不同的索引，如果访问的记录被其他session锁定，也是需要等待锁才能访问。
* 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决 定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突 时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。 
* 检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。
### 间隙锁（Next-Key锁）
当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的 索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁 （Next-Key锁）。  

举例来说，假如emp表中只有101条记录，其empid的值分别是 1,2,…,100,101，下面的SQL：Select * from  emp where empid > 100 for update;是一个范围条件的检索，InnoDB不仅会对符合条件的empid值为101的记录加锁，也会对empid大于101（这些记录并不存在）的“间隙”加锁。  

InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求，对于上面的例子，要是不使 用间隙锁，如果其他事务插入了empid大于100的任何记录，那么本事务如果再次执行上述语句，就会发生幻读；

### InnoDB事务

* InnoDB中的日志文件  
  MySQL Innodb中存在多种日志，除了错误日志、查询日志外，还有很多和数据持久性、一致性有关的日志。  

  binlog，是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的, 另外通过解析binlog能够实现mysql到其他数据源（如ElasticSearch)的数据复制。redo log 跟binary log 的区别，redo log 是存储引擎层产生的，而binary log是数据库层产生的。假设一个大事务，对tba做10万行的记录插入，在这个过程中，一直不断的往redo log顺序记录，而binary log不会记录，直到这个事务提交，才会一次写入到binary log文件中。  

  redo log，是innodb引擎层产生的日志，当数据库对数据做修改的时候，需要把数据页从磁盘读到buffer pool中，然后在buffer pool中进行修改，那么这个时候buffer pool中的数据页就与磁盘上的数据页内容不一致，称buffer pool的数据页为dirty page 脏数据，如果这个时候发生非正常的DB服务重启，那么这些数据还没在内存，并没有同步到磁盘文件中（注意，同步到磁盘文件是个随机IO），也就是会发生数据丢失，如果这个时候，能够在有一个文件，当buffer pool 中的data page变更结束后，把相应修改记录记录到这个文件（注意，记录日志是顺序IO），那么当DB服务发生crash的情况，恢复DB的时候，也可以根据这个文件的记录内容，重新应用到磁盘文件，数据保持一致。这个文件就是redo log ，用于记录 数据修改后的记录，顺序记录。

  undo Log（回滚日志）: 是innodb引擎层产生的日志，除了记录redo log外，当进行cluster index数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC。  
* 为了解决并发与隔离的矛盾，sql92 定义了4个隔离级别：  

    读未提交：会存在脏读。  
    读已提交：解决了脏读，但是仍然会有不可重复读。  
    可重复的：解决了脏读，不可重复的，但是仍有幻读问题。  
    串行执行：解决了脏读，不可重复读，幻读的问题。  

* InnoDB中怎么解决可重复读的问题呢：借助undo log实现版本隔离  
  mvcc(multiversion concurrency controll)多版本并发控制引擎。但是mvcc并不能解决幻读的问题。  
  Mysql Innodb cluster index中的行记录的存储格式，除了最基本的行信息外,还有2个字段：DB_TRX_ID,DB_ROLL_PTR  
  DB_TRX_ID（事务编号）:用来标示，最后一次对该行记录进行修改（update|insert）的事务id。  
  DB_ROLL_PTR（回滚指针）:需要回滚的那条记录的指针，指向undo log record(撤销日志记录)。如果一条记录被更新，则需要往undo log record 中插入一条被更新之前的记录。  
  此外，删除在内部被视为更新，其中行中的特殊位被设置为将其标记为已删除

* read view:数据快照，在读数据时（只有在select情况下才会创建视图），我们得到的是那个时间点的快照数据，而不用管同时其他事务对数据的修改，查询过程中，若其他事务修改了数据，那么就需要从 undo log中获取旧版本的数据。这么做可以有效避免因为需要加锁（来阻止其他事务同时对这些数据的修改）而导致事务并行度下降的问题。同时也保证了数据的一致性。  
  
* read view的数据结构，当前事务标识creator_trx_id，当前活跃事务数量n_trx_ids，当前活跃事务数组trx_ids，视图链表view_list，能看见最大版本low_limit_id（大于等于low_limit_id的版本都看不见），能看见的最小版本up_limit_id（小于up_limit_id的版本都看见）；比如：当前事务creator_trx_id=5，活跃事务n_trx_ids=4，活跃事务数组trx_ids=[8,4,3](不包括当前事务),low_limit_id=8，up_limit_id=3，视图链表view_list=trx8->trx4->trx3;在活跃数组中的事务，通过二分法查找，如果查找出来的事务与当前事务相等则可见，否则就不可见。也就是说最好可见的事务版本是当前版本和小于up_limit_id的版本。  
  
* undo record:undo记录的主要类型如下，其中TRX_UNDO_INSERT_REC为insert的undo，其他为update和delete的undo。
  ```
  #define TRX_UNDO_INSERT_REC 11 /* fresh insert into clustered index */
  #define TRX_UNDO_UPD_EXIST_REC 12 /* update of a non-delete-marked record */
  #define TRX_UNDO_UPD_DEL_REC 13 
  /* update of a delete marked record to a not delete 
  marked record; also the fields of the record can change */
  #define TRX_UNDO_DEL_MARK_REC 14 
  /* delete marking of a record; fields do not change */
  ```
  对于insert和delete，undo中会记录键值，delete操作只是标记删除(delete mark)记录。
  对于update，如果是原地更新，undo中会记录键值和老值。update如果是通过delete+insert方式进行的，则undo中记录键值，不需记录老值。其中delete也是标记删除记录。二级索引的更新总是delete+insert方式进行。
  
* 如果当前行的事务不可见，则根据DB_ROLL_PTR去undo log信息中寻找，再判断下这个版本的数据是否可见，以此类推。
  
*  何时创建快照：在RC隔离级别下，是每个SELECT都会获取最新的read view；而在RR隔离级别下，则是当事务中的第一个SELECT请求才创建read view。

* insert：当事务1插入一条数据时，回滚指针是null，生成一条Insert undo log。  
  
  update:生成一条update undo log,记录之前的旧值；DB_ROLL_PTR指向生成的update undo log  
  
  delete:生成一条update undo log,记录之前的旧值；将索引中的值标记为删除，DB_ROLL_PTR指向生成的update undo log，此时并没有真正的删除数据，而是需要purge线程来工作（补充：purge线程两个主要作用是：清理undo页和清除page里面带有Delete_Bit标识的数据行。在InnoDB中，事务中的Delete操作实际上并不是真正的删除掉数据行，而是一种Delete Mark操作，在记录上标识Delete_Bit，而不删除记录。是一种"假删除",只是做了个标记，真正的删除工作需要后台purge线程去完成。

* insert undo log是指在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

* 而update undo log记录的是对delete 和update操作产生的undo log，该undo log可能需要提供MVCC机制，因此不能再事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。


* 当事务2更新一条数据时，1、用排它锁锁定该行，2、把该行修改前的值copy到undo log中，3、修改当前值，填写事务编号，使回滚指针指向undo log修改前那条记录，

### 参考  
https://liuzhengyang.github.io/2017/04/18/innodb-mvcc/  
https://my.oschina.net/alchemystar/blog/1927425  
https://my.oschina.net/xinxingegeya/blog/505675  

## InnoDB索引
  除了spatial index(空间索引)使用的数据结果是R-tree,InnoDB的其他索引使用的物理结果都是B-Tree  

* 聚集索引（clustered index）：
  一个表只有一个聚集索引，除聚集索引外的其他索引都是非聚集索引（也叫二级索引、辅助索引）       
  如果表中存在主键，innoDB会使用主键当做聚集索引。  
  如果没有主键，innodb会使用第一个字段为非空的唯一索引当做聚集索引。  
  如果没有主键、可用的唯一索引，innodb会使用隐藏字段DB_ROW_ID生产一个聚集索引。  
  聚集索引在B-tree中叶子节点存放的是整行数据的所有值。  

* 非聚集索引  
  非聚集索引在B-tree中叶子节点存放的是索引字段的值以及指向对应的数据块一个指针（即是在聚集索引中的值）。  

  非聚集索引一般是要二次查询。但是二次查询有很费时间。  
  比如有个表存在聚集索引clustered index(id), 非聚集索引index(username)。  
  select id, username from t1 where username = '小明'  
  select username from t1 where username = '小明'  
  这两个sql语句只需要查询一次。  
  而select username, score from t1 where username = '小明' 则需要查询两次。先去非聚集索引中查询，然后再去聚集索引中查。

  组合索引，理论上可以解决一部分二次查询问题，但是组合索引需要满足最左侧索引的原则。  
