# UndoRedoLogTree
Mysql的undo, redo日志技术研究


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