# UndoRedoLogTree
Mysql的undo, redo日志技术研究

<pre>
redo log是InnoDB存储引擎层的日志；
binlog是Mysql Server层记录的日志；

两者都是记录了某些操作的日志（不是全部日志），两者有重复，两者的格式不一样。

1）
  redo log是InnoDB存储引擎层的日志，又称重做日志文件，用于记录事务操作的变化，记录的是数据修改之后的值。
     
  binlog是属于mysql server层面的，属于逻辑日志，是以二进制形式记录的是这个操作语句的原始逻辑。       

2）
  redo log是循环写，日志空间固定
  
  binlog是追加写，不会覆盖，文件变大之后切换下一个文件

3）
  binlog作为主从复制，数据恢复使用
 
  redo log作为异常宕机或者介质故障后的数据恢复使用

</pre>

<pre>
InnoDB事务日志包括redo log和undo log。
      redo log: 重做日志，提供前滚操作；
      undo log: 回滚日种子，提供回滚操作。

      undo log不是redo log的逆向操作，都是用来恢复的日志。

      redo log:
              通常是物理日志，记录的是数据页的物理修改，而不是某一行或某几行修改成什么样，
      它用来恢复提交后的物理数据页（恢复数据页，且只能恢复到最后一次提交的位置）

      undo log:
              用来回滚行记录到某个版本。 undo log一般是逻辑日志，根据每行记录进行记录。
</pre>


![](https://i.imgur.com/I49Mp7q.png)

<pre>
redo log：
         redo log包括两部分：
              1）内存中的日志缓冲（redo log buffer）
                 该部分是易失性的。
              2）磁盘上的重做日志文件（redo log file）
                 该部分是持久化的

         在概念上，innodb通过force log at commit机制实现事务的持久性，即在事务提交的时
     候，必须先将该事务的所有日志写入到磁盘上的redo log file和undo log file中进行持久化。

         为了确保每次日志都能写入到事务日志中，在每次将log buffer的日志写入日志文件的过程
     中都会调用一次操作系统的fsync操作。因为MySQL是工作在用户空间的。log buffer处于用户空
     间的内存中。要写入到磁盘的log file中，中间还要经过操作系统内核空间的os buffer，调用
     fsync的作用就是将OS buffer中的日志刷到磁盘上的log file中。
</pre>


![](https://i.imgur.com/jAkXaJH.png)

<pre>
redo log 和 undo log日志刷盘策略

     MySQL支持用户自定义在commit时如何将log buffer中的日志刷log file中。这种控制通过变量 innodb_flush_log_at_trx_commit 的值来决定。该变量有3种值：0、1、2，默认为1。

     mysql> show global variables like "innodb_log%"
     Variable_name          |           Value
     innodb_log_buffer_size	            8388608
     innodb_log_checksums	            ON
     innodb_log_compressed_pages	    ON
     innodb_log_file_size	            268435456
     innodb_log_files_in_group	        2
     innodb_log_group_home_dir	        ./
     innodb_log_write_ahead_size	    8192

     innodb_flush_log_at_trx_commit 参数可以控制将 redo buffer 中的更新记录写入到日
     志文件以及将日志文件数据刷新到磁盘的操作时机。通过调整这个参数，可以在性能和数据安全
     之间做取舍。

     0：在事务提交时，innodb 不会立即触发将缓存日志写到磁盘文件的操作，而是每秒触发一次
        缓存日志回写磁盘操作，并调用系统函数 fsync 刷新 IO 缓存。这种方式效率最高，也最
        不安全。
     1：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，并调用 fsync 
        刷新 IO 缓存。
     2：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，但并不马上调
        用 fsync 来刷新 IO 缓存，而是每秒只做一次磁盘IO 缓存刷新操作。只要操作系统不发生
        崩溃，数据就不会丢失，这种方式是对性能和数据安全的折中，其性能和数据安全性介于其
        他两种方式之间。

     innodb_flush_log_at_trx_commit 参数的默认值是 1，即每个事务提交时都会从 
     log buffer 写更新记录到日志文件，而且会实际刷新磁盘缓存，显然，这完全能满足事务的持
     久化要求，是最安全的，但这样会有较大的性能损失。

     在某些需要尽量提高性能，并且可以容忍在数据库崩溃时丢失小部分数据，那么通过将参
     数 innodb_flush_log_at_trx_commit 设置成 0 或 2 都能明显减少日志同步 IO，加快事
     务提交，从而改善性能。
</pre>

<pre>
undo log
       undo log有两个作用:
            1）提供回滚
            2）多个行版本控制（mvcc）

       在数据修改的时候，不仅记录了redo，还记录了对应的undo，如果因为某些原因导致事务失败
    了，可以借助该undo日志进行回滚。

       undo log 和 redo log不一样，undo记录的是逻辑日志，可以认为当delete 一条记录时，
    undo log中会记录一条insert记录，反之亦然，当update一条记录时，undo log记录一条对应
    相反的update记录。

       当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。有时候
    应用到行版本控制的时候，也是通过undo log来实现的；当读取的某一行被其他事务锁定时，它可
    以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定
    一致性读取。

      undo log是采用段 segment的方式来记录的，每个undo操作在记录的时候回占用一个undo 
    log segment。

      另外undo log 也会产生redo log。因为undo log也需要持久性保护。


      innodb存储引擎对undo的管理采用段的方式。rollback segment称为回滚段，每个回滚段中
    有1024个undo log segment。

      在以前老版本，只支持1个rollback segment，这样就只能记录1024个undo log segment。
    后来MySQL5.5可以支持128个rollback segment，即支持128*1024个undo操作，还可以通过
    变量 innodb_undo_logs (5.6版本以前该变量是 innodb_rollback_segments )自定义多少
    个rollback segment，默认值为128

    mysql> show variables like "%undo%"
    Variable_name               value
    innodb_max_undo_log_size	1073741824
    innodb_undo_directory	./
    innodb_undo_log_truncate	OFF
    innodb_undo_logs	128
    innodb_undo_tablespaces	0
</pre>

<pre>
调整 innodb_log_buffer_size

    innodb_log_buffer_size 决定 innodb 重做日志缓存池的大小，默认是 8MB。对于可能产生
    大量更新记录的大事务，增加 innodb_log_buffer_size 的大小，可以避免 innodb 在事务
    提交前就执行不必要的日志写入磁盘操作。因此，对于会在一个事务中更新，插入或删除大量记录
    的应用，可以通过增大 innodb_log_buffer_size 来减少日志写磁盘操作，提高事务处理性能。
</pre>

<pre>
设置 log file size ，控制检查点

      当一个日志文件写满后，innodb 会自动切换到另一个日志文件，但切换时会触发数据库检查
      点(checkpoint),这将导致 innodb 缓存脏页的小批量刷新，会明显降低 innodb 的性能。

      可以通过增大 log file size 避免一个日志文件过快的被写满，但如果日志文件设置的过
      大，恢复时将需要更长的时间，同时也不便于管理，一般来说，平均每半个小时写满一个日志
      文件比较合适。

      可以通过下面的方式来计算 innodb 每小时产生的日志量并估算合适的 innodb_log_file_size 的值：
            // 1. 计算 innodb 每分钟产生的日志量  
			MySQL [(none)]> pager grep -i "log sequence number"
			PAGER set to 'grep -i "log sequence number"'
			
			MySQL [(none)]> show engine innodb status\G select sleep(60);show engine innodb status\G
			Log sequence number 1706853570
			1 row in set (0.00 sec)
			
			1 row in set (1 min 0.00 sec)
			
			Log sequence number 1708635750
			1 row in set (0.00 sec)
			
			MySQL [(none)]> nopager
			PAGER set to stdout
			
			MySQL [(none)]> select round ((1708635750 - 1706853570) /1024/1024) as MB;
			+------+
			| MB   |
			+------+
			|    2 |
			+------+
			1 row in set (0.00 sec)

       通过上述操作得到 innodb 每分钟产生的日志量是 2 MB。然后计算没半小时的日志量
       半小时日志量 = 30 * 2MB = 60MB
       这样，就可以得出 innodb_log_file_size 的大小至少应该是 60MB。
</pre>

![](https://i.imgur.com/BCb4xd5.png)

