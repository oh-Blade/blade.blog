# 磁盘
在计算机中，数据时存储在磁盘里的，那么以磁盘的维度它有两个指标，一个是寻址，寻址的速度是毫秒级的，第二个是带宽，即单位时间有多少字节流过去，基本上是G或者M这种级别。
除了磁盘还有内存，内存的寻址是纳秒级别。1秒=1000毫秒=1000 * 1000微妙=1000 * 1000 * 1000纳秒，显然如果数据在内存中，找到它放到CPU一定是非常快的，相比磁盘快十万甚至百万倍。

磁盘有[磁道](https://baike.baidu.com/item/%E7%A3%81%E9%81%93)和[扇区](https://baike.baidu.com/item/%E6%89%87%E5%8C%BA/3642285)，一个扇区512个字节。如果我们访问一个数据时，磁盘是以最小力度以一个扇区一个扇区来找，那么对于一块普通硬盘1T或者2T，就会有很多很多个512字节，而我们要找的数据在哪块512字节放着呢？这样就会导致索引成本很大。也就是说如果一个1t硬盘里面都是512字节的小格子的话，那么上层操作系统当中就得准备一个索引，这个索引就不是4个字节了，可能是8个字节或者更多，因为他要一个能表示很大数的区间，才能索引出这么多的512小格子。其实真正使用硬件的时候并不是按照512字节为一次读写量，一般默认格式化4K操作系统，无论读多少都是最少拿4k
![image.png](https://www.hounk.world/upload/2021/01/image-be833047c29b4cce9c1101aadcf4718c.png)
有个以上结论，如果数据放在文件里的话，随着文件变大，他的查询一定会变慢，因为访问硬盘的时候会受到硬盘瓶颈的影响，也就是I/O称为瓶颈。

# 数据库
如果数据都是以文件直接存在磁盘，肯定会很慢。为了改变这种情况，随着技术的发展出现了数据库。
第一，数据库有了data page的概念，他的data page大小是4K（不同数据库不同）这个4K就和磁盘的4k刚好对上。曾经数据是线性存在一个文件里面，连起来的不能割开，不容易找到。但是现在如果数据库准备了很多这样的4k，在软件里面先定义出一个4k，然后每个4k有自己的ID，而且要读取的数据正好符合磁盘的一次IO。如果把数据库的data page设置为1k，就会出现上层数据库软件想读1k但是硬盘还是读4k，索性还不如直接设置为4k。当然也可以设置为8k，16k，但是调小了其实是浪费。那么如果只有这样一个个4k的格子，查找数据的成本复杂度还是没有变化，还是需要从第一个4k读到内存，挨个去找，这样还是很慢。
第二，数据库通过通过建表建索引来提速。通过指定索引列，例如用id字段建索引，每个id指向哪一个data page。
第三，在建数据表的时候需要给出schema，就是必须给出一共多少列，每个列的类型是什么，约束是什么。那么每一个列的类型其实就是字节宽度，只要schema给出类型那么这个表立案每一行的数据宽度就定死了。如果向这个表插入一条数据，假设这一行有10个字段，只给出了前两个，但是向data page存的时候，后面八个字段都会用0去开辟，这样以后在增删改的时候不用移动数据，直接在那个位置复写就可以了。
## 数据库的表很大，性能一定降低？
1. 针对增删改，那么性能会降低，因为你必须去修改索引或调整它的位置。
2. 如果是一条或者少量查询，而且能够命中索引，这个时候where条件走的还是内存B+Tree，走的还是一个索引块，这个索引块到内存依然走的是一个data page，查询依然很快； 如果是并发量大的时候，这就不是要获取一个data page到内存了，因为查询数量变大，被命中的数据量变大，假如有一万个查询，每个查询获得一个4k数据，查询条件不一样，，刚好散货在不同的4k上，这一万个查询进入到服务器之后，每个查询的4k需要挨个进入到内存，这就会出现部分插叙需要等待。 硬盘的慢，除了寻址慢，还有贷款的限制
# memcache
在redis之前已经有了memcache，也是key-value的，但是redis出来后反而取代了memcache的地位。
本质区别是因为memcache的value是没有类型概念的。如果报保存复杂数据可以使用json。那么redis的类型概念有什么优势呢？
1. 如果客户端想在kv的缓存系统中取出value的其中一个数据，也就是memcache的value保存了一个数组，这时候就不得不返回value的所有数据到client，那么当客户端很多时网卡的IO就会变成最大的瓶颈。
2. 取回数据之后，客户端要自己实现解码，从json中解码出所要数据。

如果时redis呢？因为redis支持类型，value直接存放list数据，客户端只需要调用redis的api，传入一个下标或者lpop等等各种命令就可以获取到目标数据，客户端不需要写很多解析的代码，而且还减小了redis服务端网卡的压力。

其实这就是计算向数据移动。因为客户端不是把数据拿回来再发生计算，而是调到他的方法，计算直接在那边发生，返回目标数据。
