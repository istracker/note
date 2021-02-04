## 前言
--- 高可用
在web服务器中，高可用是指服务器可以正常访问的时间，衡量的标准是在多长时间内可以提供正常服务。
在Redis语境中，高可用除了保证服务器正常服务之外，还需要考虑数据容量的拓展、数据安全不丢失等。

在Redis中，实现高可用用的技术主要包括 `持久化`、`复制(读写分离)`、`哨兵`和`集群`。

下面分别介绍它么的作用(解决什么样的问题)以及如何实现.

---


## 1. 持久化
持久化是最简单的高可用方法，有时甚至不被归为高可用的手段。主要作用是数据备份，即将数据存储到硬盘，保证数据不会因进程的退出而丢失。

### <1> Redis持久化概述
持久化功能：
	Redis是内存数据库，数据都是存储在内存中的，为了避免进程退出导致数据的永久丢失，需要定期将Redis中的数据以某种形式(数据或命令)从内存保存到磁盘。当下次Redis重启时，
利用持久化文件实现数据恢复。除此之外，为了进行灾难备份，可以将持久化文件拷贝到一个远程位置。

Redis持久化实现分为 RDB持久化 和 AOF持久化，前者将当前数据保存到磁盘，后者则是将每次执行的写命令以append追加的形式保存到磁盘(类似于MySQL的Binlog)。
由于AOF持久化的实时性更好，即当进程意外退出时，丢失的数据更少，因此AOF是目前主流的持久化方式，不过RDB持久化仍然有其用武之地。


### <2> RDB持久化
RDB持久化是将当前进程中的数据生成快照保存到磁盘(因此也被称作快照持久化)，保存的文件后缀是.rdb；当Redis重新启动时，可以读取快照文件恢复数据。

#### 触发条件：

. save m n
m秒内至少发生了n次写命令。（是通过serverCron函数、dirty计数器、和lastsave时间戳来实现的）

. bgsave：
在主从复制场景下，如果从节点执行全量复制操作，则主节点会执行bgsave命令，并将rdb文件发送给从节点；
执行shutdown命令时，自动执行rdb持久化，

主进程会fork出一个子进程用来执行RDB持久化过程
bgsave的实现过程：
 .1 Redis父进程首先判断：当前是否在执行save，或bgsave/bgrewriteaof（后面会详细介绍该命令）的子进程，如果在执行则bgsave命令直接返回。bgsave/bgrewriteaof 的子进程不能同时执行，主要是基于性能方面的考虑：两个并发的子进程同时执行大量的磁盘写操作，可能引起严重的性能问题。
 .2 父进程执行fork操作创建子进程，这个过程中父进程是阻塞的，Redis不能执行来自客户端的任何命令；
 .3 父进程fork后，bgsave命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令；
 .4 子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换；
 .5 子进程发送信号给父进程表示完成，父进程更新统计信息。



RDB文件是经过压缩的二进制文件.
Redis默认采用LZF算法对RDB文件进行压缩。虽然压缩耗时，但是可以大大减小RDB文件的体积，因此压缩默认开启；
需要注意的是，RDB文件的压缩并不是针对整个文件进行的，而是对数据库中的字符串进行的，且只有在字符串达到一定长度(20字节)时才会进行。

#### 启动时加载
RDB文件的载入工作是在服务器启动时自动执行的，并没有专门的命令。
但是由于AOF的优先级更高，因此当AOF开启时，Redis会优先载入AOF文件来恢复数据；只有当AOF关闭时，才会在Redis服务器启动时检测RDB文件，并自动载入。
服务器载入RDB文件期间处于阻塞状态，直到载入完成为止。
Redis载入RDB文件时，会对RDB文件进行校验，如果文件损坏，则日志中会打印错误，Redis启动失败。



#### RDB常用配置
下面是RDB常用的配置项，以及默认值：
save m n：bgsave自动触发的条件；如果没有save m n配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发。

stop-writes-on-bgsave-error yes：当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no。

rdbcompression yes：是否开启RDB文件压缩。

rdbchecksum yes：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现。

dbfilename dump.rdb：RDB文件名。

dir ./：RDB文件和AOF文件所在目录。


> ################################ SNAPSHOTTING  ################################
> #
> # Save the DB on disk:
> #
> #   save <seconds> <changes>
> #
> #   Will save the DB if both the given number of seconds and the given
> #   number of write operations against the DB occurred.
> #
> #   In the example below the behaviour will be to save:
> #   after 900 sec (15 min) if at least 1 key changed
> #   after 300 sec (5 min) if at least 10 keys changed
> #   after 60 sec if at least 10000 keys changed
> #
> #   Note: you can disable saving completely by commenting out all "save" lines.
> #
> #   It is also possible to remove all the previously configured save
> #   points by adding a save directive with a single empty string argument
> #   like in the following example:
> #
> #   save ""
> 
> save 900 1
> save 300 10
> save 60 10000
> 
> # By default Redis will stop accepting writes if RDB snapshots are enabled
> # (at least one save point) and the latest background save failed.
> # This will make the user aware (in a hard way) that data is not persisting
> # on disk properly, otherwise chances are that no one will notice and some
> # disaster will happen.
> #
> # If the background saving process will start working again Redis will
> # automatically allow writes again.
> #
> # However if you have setup your proper monitoring of the Redis server
> # and persistence, you may want to disable this feature so that Redis will
> # continue to work as usual even if there are problems with disk,
> # permissions, and so forth.
> stop-writes-on-bgsave-error yes
> # AUTH <password> as usually, or more explicitly with AUTH default <password>
> # if they follow the new protocol: both will work.
> #
> # requirepass foobared
> 
> # Command renaming (DEPRECATED).
> #
> # ------------------------------------------------------------------------
> # WARNING: avoid using this option if possible. Instead use ACLs to remove
> # commands from the default user, and put them only in some admin user you
> # create for administrative purposes.
> # ------------------------------------------------------------------------
> #
> # It is possible to change the name of dangerous commands in a shared
> # environment. For instance the CONFIG command may be renamed into something
> # hard to guess so that it will still be available for internal-use tools
> # but not available for general clients.
> #
> # Example:
> #
> # rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
> #
> # It is also possible to completely kill a command by renaming it into
> # an empty string:
> #
> # rename-command CONFIG ""
> #
> # Please note that changing the name of commands that are logged into the
> # AOF file or transmitted to replicas may cause problems.




### <3> AOF持久化



---
































## 2. 复制
复制是实现Redis高可用的基础，哨兵和集群都是在复制的基础上实现高可用的。复制主要实现了数据的多机备份以及对`读操作`的负载均衡和简单的`故障恢复`。
缺陷是故障无法自动化，`写操作`无法负载均衡，`存储能力`受到单机的限制。











## 3. 哨兵
在复制的基础上，哨兵实现了自动化的故障恢复。
缺陷是`写操作`无法负载均衡，存储能力受到单机的限制。





## 4. 集群
通过集群，Redis解决了`写操作`无法负载均衡以及存储能力受到单机限制的问题，实现了较为完善的高可用方案。