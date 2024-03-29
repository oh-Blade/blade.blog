# 1 RDB
在指定的时间间隔内将内存中的数据集快照写入磁盘，它恢复时是将快照文件直接读到内存里。

Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到 一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。 整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能 如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效，但是最后一次持久化后的数据可能丢失。

## 1.2 优点
1. 按照不同时间点保存不同版本；
2. 是一个紧凑的文件，便于灾难恢复；
3. 最大限度提高redis的性能，父进程只是分派子进程进行磁盘IO操作
4. 与AOF相比，允许大型数据集更快地恢复

## 1.3 缺点
1. 部分数据可能会丢失；
2. 需要fork()才能使用子进程进行持久化工作，如果数据集很大，for()可能会很耗时。
## 1.4 save和bgsave
### 1.4.1 save
save的问题在于是同步阻塞的，如果是停机维护数据可以使用这种方式，但真实使用到的时候很少。一般情况下是使用bgsave
### 1.4.2 bgsave
与save不同的是bgsave是异步、非阻塞的。可以一边进行持久化，一边对外提供读写服务，互不影响，新写的数据对持久化不会造成数据影响，持久化后会将新的rdb文件覆盖之前的。
通过bgsave命令或当配置文件的条件触发时会执行。
> 注：虽然配置文件中是使用的save，但其实是完成的bgsave的操作

在 redis.conf 配置文件中的 SNAPSHOTTING 下
![image.png](https://www.hounk.world/upload/2021/01/image-71f4eb1718cd48a7ae4f5f950cc2dcf8.png)
```
save 900 1：表示900 秒内如果至少有 1 个 key 的值变化，则保存
save 300 10：表示300 秒内如果至少有 10 个 key 的值变化，则保存
save 60 10000：表示60 秒内如果至少有 10000 个 key 的值变化，则保存
```
如果redis只是作为缓存，不需要持久化，可以把以上注释调，用save ""表示停用。

#### 1.4.2.1 原理fork
简单来说就是redis通过调用fork函数，来启动一个子进程执行持久化操作。fork出子进程后，和父进程的数据是隔离的，父进程的增删改都不会影响子进程，父进程也不会因为子进程持久化失败而出现问题。

fork是什么？参考[《百科》](https://baike.baidu.com/item/fork/7143171)

当bgsave执行时，Redis主进程会判断当前是否有fork()出来的子进程，若有则忽略，若没有则会fork()出一个子进程来执行rdb文件持久化的工作，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，此时如果父进程对数据进行修改。这个时候就会使用操作系统的 COW （Copy On Write）机制来进行数据段页面的分离。数据段是由很多操作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。
#### 1.4.2.2 原理copy on write
[《跳转知乎》](https://zhuanlan.zhihu.com/p/136428913)

## 1.5 恢复数据
将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可，redis就会自动加载文件数据至内存了。Redis 服务器在载入 RDB 文件期间，会一直处于阻塞状态，直到载入工作完成为止。

# 2 AOF

## 2.1 优势
1. 可以具有不同的 fsync 策略：no/everysec/always。默认策略是everysec，每秒钟写入性能仍然很大（fsync 使用后台线程执行，并且主线程在未进行 fsync 时将尝试尝试执行写入），但您只能丢失一秒钟的写入。
* no的意思就是在有增删改的命令时，每个操作都会向buffer写，但是redis不调flush，内核什么时候满了什么时候刷，这样可能会丢失一个buffer的数据
* everysec是每秒调一次flush，最多可能会丢失接近一个buffer的数据
* always是只要写了一个文件描述符，就把这个文件描述符写进去，并同时调一个flus写入磁盘，所以always一定是最可靠的

2. AOF 日志是仅追加日志，因此，如果发生断电，则没有查找或损坏问题。即使由于某种原因（磁盘已满或其他原因），日志以半写命令结束，重做检查工具也能够轻松修复它。
3. Redis 能够自动在后台重写 AOF，当它变得太大。重写是完全安全的，因为当 Redis 继续追加到旧文件时，将生成一个全新的重写，只需创建当前数据集所需的最小操作集，一旦第二个文件准备就绪，Redis 将切换两个文件并开始追加到新数据集。
4. AOF 以易于理解和解析的格式，一个又一个地包含所有操作的日志。甚至可以轻松地导出 AOF 文件。例如，即使使用 FLUSHALL 命令刷新了所有错误，如果在此期间未执行日志重写，您仍然可以保存数据集，只需停止服务器、删除最新命令并再次重新启动 Redis。
## 2.2 缺点
1. AOF 文件通常大于同一数据集的等效 RDB 文件。
2. AOF 可能比 RDB 慢，具体取决于确切的 fsync 策略。通常，将 fsync 设置为每一秒的性能仍然很高，禁用 fsync 时，即使在高负载下，其速度也应该与 RDB 完全一样快。即使写入负载巨大，RDB 仍能够提供有关最大延迟的更多保证。
# 3 混合式
从redis4.0开始，提供了aof和rdb混合的方式，集中了两者的有点。
[《跳转简书》](https://www.jianshu.com/p/446b12e4740f)







参考：
https://redis.io/topics/persistence
https://www.redis.com.cn/redis-persistence.html
https://zhuanlan.zhihu.com/p/136428913
https://www.zhihu.com/question/303765035/answer/1255056210
