# Redis持久化方式
## RDB 
```text
在指定时间间隔内，将内存中的数据集快照写入磁盘，也就是Snapshot快照，它恢复时将快照文件直接读到内存之中。来恢复数据。
```
持久化
```text
Redis单独创建（fork）一个子进程来进行持久化，现将数据写入一个临时文件之中，等到持久化过程结束来，再用这个临时文件
替换上次持久化的文件好的文件。在这个过程之中，只有子进程来负责IO操作，住进程仍然处理客户端请求
默认情况下，Redis将数据库快照保存在名字为dump.rdb的二进制文件中。通过触发快照形式，来做到指定时间间隔内的
数据持久化道dump.rdb
```
在redis.conf配置文件中
```text
#   In the example below the behaviour will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#

save 900 1 # 900s至少1个key更新 则触发快照
save 300 10 # 300s至少10个key更新 则触发快照
save 60 10000 # 60s至少10000个key更新 则触发快照

# The filename where to dump the DB
dbfilename dump.rdb # 快照文件
```
优点
```text
1.大数据集恢复，RDB方式要比AOF方式恢复速度快
2.RDB可以最大化Redis性能，父进程做的是fork子进程，然后继续处理客户端请求，让子进程负责持久化操作，父进程无需进行IO操作
3.RDB是一个非常紧凑的文件，通过备份原文到本机一外的其它主机上，一单发生宕机，就能将备份文件复制到Redis安装目录下，通过启动服务就能王成数据到恢复
```
缺点
```text
1.RDB持久化方式不适合对数据完整性要求很高对情况，因为，尽管我们可以修改快照频率触发快照，但是有可能会宕机，
导致数据并没有持久化入快照文件之中
2.每一次RDB时，父进程都会fork一个子进程，优子进程负责持久化操作，如果数据集庞大，fork出子进程过程就会很耗时
```

## AOF
```text
以日志对形式记录Redis每一个写操作，将Redis执行过的所有写命令记录下来，只许追加文件不可以改写文件，Redis启动之后
会读取appendonly.aof文件来实现重新恢复数据，默认不开启，需要将redis.conf中的appendonly  no改为yes
```
配置文件如下
```text
appendonly yes # 开启AOF同步

# The name of the append only file (default: "appendonly.aof")

appendfilename "appendonly.aof"

# The fsync() call tells the Operating System to actually write data on disk
# instead of waiting for more data in the output buffer. Some OS will really flush
# data on disk, some other OS will just try to do it ASAP.
#
# Redis supports three different modes:
#
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
#
# The default is "everysec", as that's usually the right compromise between
# speed and data safety. It's up to you to understand if you can relax this to
# "no" that will let the operating system flush the output buffer when
# it wants, for better performances (but if you can live with the idea of
# some data loss consider the default persistence mode that's snapshotting),
# or on the contrary, use "always" that's very slow but a bit safer than
# everysec.
#
# More details please check the following article:
# http://antirez.com/post/redis-persistence-demystified.html
#
# If unsure, use "everysec".

# appendfsync always # 每修改同步，每一次发生数据变更都会持久化到磁盘上，性能较差，但数据完整性较好。
appendfsync everysec # 默认每秒同步一次，每秒内记录操作，异步操作，如果一秒内宕机，有数据丢失。
# appendfsync no # 不同步
```
数据恢复
```text
重启Redis时，如果dump.rdb与appendonly.aof文件同时存在时，Redis会自动读取appendonly.aof文件来恢复数据，
如果appendonly.aof文件破损，可以使用redis-check-aof工具来修复
如果dump.rdb文件有破损，我们也可以用redis-check-rdb工具来修复
```
重写
```text
当然如果AOF 文件一直被追加，这就可能导致AOF文件过于庞大。因此，为了避免这种状况，Redis新增了重写机制，
当AOF文件的大小超过所指定的阈值时，Redis会自动启用AOF文件的内容压缩，只保留可以恢复数据的最小指令集，可以使用命令bgrewiteaof。
重写原理：AOF文件持续增长过大时，会fork出一条新进程来将文件重写（也是临时文件最后再rename）,遍历新进程的内存中的数据，
每条记录都会有一条set语句，重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，有点类似于快照。
```
优点
```text
Redis可以在AOF文件变得过大时，会自动地在后台对AOF进行重写：重写后的新的AOF文件包含了恢复当前数据集所需的最小命令集合。
整个重写操作是绝对安全的，因为Redis在创建AOF文件的过程中，会继续将命令追加到现有的AOF文件中，即使在重写的过程中发生宕机，
现有的AOF文件也不会丢失。一旦新AOF文件创建完毕，Redis就会从旧的AOF文件切换到新的AOF文件，并对新的AOF文件进行追加操作。

AOF文件有序地保存了对数据库执行的所有写入操作。这些写入操作一Redis协议的格式保存，易于对文件进行分析；
例如，如果不小心执行了FLUSHALL命令，但只要AOF文件未被重写，通过停止服务器，移除AOF文件末尾的FLUSHALL命令，重启服务器就能达到FLUSHALL执行之前的状态。
```

缺点
```text
对于相同的数据集来说，AOF文件要比RDB文件大。
根据所使用的持久化策略来说，AOF的速度要慢与RDB。一般情况下，每秒同步策略效果较好。不使用同步策略的情况下，AOF与RDB速度一样快。
```


