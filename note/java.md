# Mysql

 

## 原理

### 客户端和服务端

​                                                  

 

#### 连接管理

客户端进程采用TCP/IP、共享内存、Unix套接字几种方式来连接服务器。服务器创建一个线程来与其建立通信。当客户端退出时候，线程不会立即销毁，而是缓存起来以备下次使用。

 

客户端发起请求时候需要携带认证信息。当不再一台机上时候，可以采用SSL（安全套接字）来建立通信保障安全。

 

#### 解析与优化

##### 查询缓存

Mysql会把第一次请求进来的查询请求缓存下来，如果在没有对其进行增删改时候，且没有对语句进行改变并且没有使用函数，那么下次将直接返回结果给客户端。

##### 语法解析

对文本进行解析，具体过程就像jvm对class文件进行解析过程类似，就是语法语义的解析。

##### 查询优化

优化生成一个执行计划，表明了应该使用那些索引，表之间连接关系。可以使用Explain对某个语句查看执行计划。

 

查询执行方法分为全表扫描和索引查询。

 

###### 访问方法

把mysql执行查询语句的方式叫访问方法

 

Const

通过主键或者唯一二级索引列与常数的等值比较来定位叫做Const。

但唯一二级索引列与null比较的时候不会采用该方法

Ref

普通二级索引列与常数等值比较。当匹配记录少的时候采用索引查询，记录过多的时候全表扫描

 

注意两种情况：

1、 不论是普通还是唯一二级索引，它们索引列对包含null值的数量并不限制。Key is null的搜索条件最多是ref

2、 多于联合索引，最左边的连续索引列与常数的等值比较可能会采用ref

Ref_or_null

SELECT * FROM single_demo WHERE key1 = 'abc' OR key1 IS NULL;

这种访问方法就是采用ref_or_null

 

Range

SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);

利用索引进行范围匹配的访问。

 

Index

SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';

由于该方法返回列值为联合索引的列，并且查询条件也在联合索引列中，所以遍历二级索引的叶子节点。

All

就是全表扫描

 

###### 索引合并

Intersection（交集）合并

SELECT * FROM single_table WHERE key1 = 'a' AND key3 = 'b';

为什么会使用两个索引一起查询呢，不是性能应该比一个索引低吗？

因为二级索引是顺序I/O而聚族索引是随机I/O，如果是一个索引的话，回表数据会很大，但如果两个索引一起求出主键，并且算出交集再去回表可以减少很多不必要的回表数据。

两种情况可能用它：

1、 二级索引列是等值匹配，对于联合索引，每个列都必须等值匹配，不能出现匹配部分列

2、 主键列可以使范围查询

 

Union并集

SELECT * FROM single_table WHERE key1 = 'a' OR key3 = 'b'

有三种情况用它：前两种去交集一样，第三种是使用交集索引合并的搜索条件

 

SELECT * FROM single_table WHERE key1 = 'a' AND key3 = 'b';

不过上面的语句之所以采用Intersection，归根结底还是key1和key3不是联合索引，如果换成联合索引就直接采用联合索引了。也是就是ref访问方法

 

###### 基于规则优化

Mysql帮助把用户写出的比较糟糕的语句转换为较为高效执行的语句。

条件化简

1、 常量传递：a = 5 AND b > a 转换为a = 5 AND b > a

2、 等值传递：a = b and b = c and c = 5 转换为a = b and b = c and c = 5

3、 移除没用的条件：对于⼀些明显永远为TRUE或者FALSE的表达式，优化器会移除掉它们

4、 表达式计算：a = 5 + 1转换为 a=6

 

子查询语法注意事项

1、 子查询必须用括号括起来

2、 在SELECT子句中的子查询必须是标量查询（也就是只返回一个结果）

3、 在想得到标量子查询或者行子查询，但不能保证子查询结果集只有一条，可以使用Limit 1限制

4、 对于IN/ANY/SOME/ALL 子查询中不允许有LIMIT语句，同时ORDER BY和DISTINCT BY和GROUP BY也同样没用

 

优化

Mysql会把子查询结果集中的记录存到临时表的过程称为物化。

如果数据不多，就采用内存（哈希索引）

如果数据多，就采用磁盘（B+数索引）

 

之后与外表转换成内连接，之后根据实际来选择谁为驱动表，谁不为驱动表。

 

当然也有其他情况：

不进行物化，直接转换为连接。叫半连接，是mysql内部使用的，为了消去重复，不用建立临时表！！！

 

如果In子查询不符合半连接要求，那么就会根据情况从下面两个选择一种

1、 先把子查询物化之后查询

2、 执行IN转为EXISTS，这样可以使用索引（一般IN在WHERE或者ON之后） 

 

 

From子句里的叫派生表。它首先会看是否能与外表合并，如果不行再物化派生表。

 

 

 

##### 存储引擎

Mysql服务器把数据的存储和提取封装到一个存储引擎的模块。

 

 

 

###### Innodb

将数据划为若干个页，以页作为磁盘和内存之间交互的基本单位，页大小一般为16k。

 

行格式

Compact

   

1、 变长字段长度列表：把所有变长字段的真实数据所占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表，按列的逆序排序。他不存储列值为null的列。

2、 Null值列表：存储列值为null的列，也是按列的顺序逆序排列。

3、 记录头信息：固定为5个字节，

4、 记录的真实数据：除了列值以为，mysql会创建一些隐藏列：

DB_ROW_ID唯一标识一条记录

DB_TRX_ID 事务id

DB_ROLL_PTR 回滚指针

 

Innodb表主键生成策略：优先使用用户自定义的主键作为主键，如果没有，选择一个Unique作为主键，还是没有的话，添加隐藏列row_id作为主键。

 

对于char(M)如果采用的是定长字符集则不加入到变长字段长度列表，如果不是就要添加到变长字段长度列表里面。

 

 

 

使用varchar、text、blob造成的行溢出。该页只记录真实数据的前780左右个字节，剩下的存储其他数据所在的页，其他页会通过链表连接起来。其他页称为溢出页。

   

 

 

Dynamic和Compressed

与Compact大致相同，只是如果溢出了，就会把存储真实数据的位置只存储溢出页的位置。

 

Compressed采用压缩算法对页面进行压缩，节省了空间。

 

 

数据页

   

FreeSpace就是可用空间，当要保存数据时候，就从FreeSpace申请空间用来保存数据。

 

 

 

 

   

1、 delete_mask:当前记录是否删除。被删除的记录不会立即从磁盘中删除。因为移除他们需要重新排序消耗性能，而是做个标记，形成垃圾链表。

2、 heap_no当前记录再页中的位置，0和1被虚拟记录给占用了，分别是最小和最大记录。

   

3、 next_recod：当前记录的真实数据到下一条记录的真实数据的地址偏移量。下一条记录是按照主键从小到大的下一条数据。Infimum（最小纪录）下一条记录就是主键值最小的用户记录。

 

 

 

PageDirectory

   

 

在数据页中查找指定主键值的记录过程分为：

1、 通过二分法确定该记录所在的槽。并找到该槽最小记录

2、 通过记录next_record属性遍历该槽所在组的各个记录

 

PageHeader

记录槽数量、记录数量等

 

 

FileHeader

页编号、上下页页号、页类型等等。

索引页通过双向链表连接。

   

 

 

页和记录的关系：查找某条记录，先根据主键在页目录二分查找法找到对应的页。在根据二分查找对应的槽，在槽中遍历记录来找到对应记录。

   

 

索引

如果当前查询条件不是主键查询，那么在没有索引的情况下，就只能从最小纪录遍历到最大记录查找对应记录了。

 

   

也采用页来存储目录项也就是索引。怎么区分这些页，通过页的record_type，为1就是目录项（索引）记录，0就是普通用户记录。

目录项记录只有主键值和页号两个列

而这种数据结构就叫B+树！！！

我们实际的数据存储在叶子节点上，其余都叫非叶子节点。

 

聚族索引

特点：

1、 使用记录主键值大小进行记录和页的排序

2、 叶子节点存储的是完整的用户记录

 

Innodb存储引擎会自动为我们创建聚族索引

 

二级索引

特点：

1、 使用对应列大小进行记录和页的排序

2、 叶子节点不存储完整用户记录（该列和主键）

3、 目录项纪录不再是主键+页号，而是该列和页号

 

如果要找到对应列数据，需要先找到该列对应的主键，再去根据主键去主键b+树上找到完整数据。这个过程叫回表

 

联合索引

以两个列以上建立的二级索引叫联合索引。

 

 

 

MyIsam索引方案：

索引和数据分开。表中记录按顺序存储来文件中，称之为数据文件。可以通过行号快速查找。不能使用二分法查找。

索引信息存储到一个称为索引文件的地方。索引叶子节点是主键+行号。查到主键后，根据行号来找到数据。所以MyIsam的索引都是二级索引。

 

 

注意事项

排序

1、 如果我们想使用联合索引，搜索条件中的列必须是最左边且连续的列。否则无法使用索引。

2、 使用联合索引排序必须按照索引列的顺序给出。

3、 当联合索引左边的值为常数时候，可以使用右边的列进行排序

4、 在排序的时候，如果where子句使用了不是索引列的列，那么排序无法使用索引

5、 排序的时候对排序列使用了函数表达式，也不能使用索引排序。

 

分组

也可以按照索引的顺序进行分组

 

回表的代价：

二级索引查出来的主键值去聚族索引中查找对应数据会相较于二级索引查询慢。如果回表的数据过多就会造成性能低，这时候还不如不适用二级索引而采用全表扫描。

 

什么时候采用全表什么时候二级索引+回表？

查询优化器根据计算来判断！！！

 

覆盖索引：

查询列表中只包含索引列，这时候就不会回表去查找数据了。不鼓励使用*来作为查询列表

 

如何挑选索引？ 

1、 只为用于搜索、排序、分组的列添加索引

2、 最好为基数大（列值不一样的比例）的列建立索引，因为基数小的，列值不够分散，都聚拢在一起的

3、 索引列类型尽量小

数据类型越小，查询时候进行比较操作越快

数据类型越小，索引占用存储空间越小，数据页中可以存储更多纪录，从而减少磁盘I/O性能消耗

4、 索引列在表达式中单独出现

5、 按照主键大小插入顺序

所以建议主键使用自增

6、避免列重复索引

 

 

#### 数据库与文件系统

##### 数据目录和安装目录

Mysql服务器启动时候会到文件系统某个目录下加载一些文件，之后在运行过程中产生的数据都会存储到这个目录下的某些文件中，这个目录就叫数据目录。

 

表在文件系统中如何存储？

表信息一般有表结构的定义和表中的数据

表结构定义就是多少列，列类型、约束条件、索引和编码字符集等

会在对应的数据库下创建一个表名.frm文件来存储

 

表中数据怎么存储？

InnoDb和MyIsAM不一样

**InnoDb**的数据就存在表空间里，表空间又分为几种

1、 系统表空间：从MySQL5.5.7到MySQL5.6.6默认下创建一个叫Ibdata1的文件存储数据，可以自动扩展大小

2、 独立表空间：MySQL5.6.6以后，InnoDb不会默认把表数据存到系统表空间，而是为每一个表创建独立表空间，名字为表名.ibd

当然可以自己使用哪个表空间，有两种方法

1、启动参数innodb_file_per_table，为0时候使用系统表空间，1为独立表空间

但改参数只对新建表有用

2、如果我们想把已经存在系统表空间 中的表转移到独⽴表空间，可以使⽤下边的语法： ALTER TABLE 表名 TABLESPACE [=] innodb_file_per_table; 

或者把已经存在独⽴表空间的表转移到系统表空间，可以使⽤下边的 语法： ALTER TABLE 表名 TABLESPACE [=] innodb_system;

 

**MyIsam**表没有表空间概念，并且数据和索引是分开存放在对应数据库中的

对应就是表名.frm

​                  表名.MYD 表数据

​                  表名.MYI           表索引

 

###### 表空间详解

**区**：每个区默认64个页组成

下面的page State bitmap每两个bit位代表一个页的页号

   

**组**：每256个区划分为一个组，每个组最开始的几个页面类型是固定的

   

我们看到每个表空间第一个组的第一区的第一个页面有个叫FSP_HDR。他存储了段直属于表空间的三个链表和存放INODE页的链表。

   

**链表基节点**：ListBase Node，

记录了链表的头尾指针和节点数。

   

 

**段**：每个段是某些零碎页面以及一些完整区的集合。（每个索引两个段，一个是叶子节点另一个是非叶子节点）

存储了FREE、NOT_FULL、FULL链表的基节点

   

 

如何知道每个段对应的INODE Entry？

在Index类型页上，pageHeader部分有下面两个重要信息

   

而上面的信息又对应了一个叫Segement Header的结构

所以PAGE_BTR_SEG_LEAF就记录了叶子节点对应的INODE ENTRY在哪个表空间的哪个页面的具体位置。并且一个索引只有两个段，所以在索引的根页面记录下这两个结构的段即可。

   

 

 

表空间由如干个区组成，每个区对应一个XDES Entry结构，直属于表空间的区对应的XDES Entry结构可以分成FREE、FREE_FLAG、FULL_FLAG这三个链表。每个段可以附属若干区，每个段上的区可以分分为FREE、NOT_FULL、FULL这三个链表。每个链表对应一个ListBase Node，

记录了链表的头尾指针和节点数。

 

数据字典：

Mysql如何知道某个表有多少列，每个列有多少字段，字段的类型呢？

通过内部系统表来获取这些信息

   

那怎么获取这些这些表数据呢？

Innodb通过固定页面记录这四个表的聚族索引和二级索引组成的B+树名字叫DataDirector Header

 

##### 系统数据库

\1.      mysql：核心数据库，存储了用户账户和权限信息等

\2.      information_schema：保存维护的表信息视图等

\3.      performance_schema：性能监控

\4.      sys：通过视图形式把前两个数据库结合起来

 

### join

对于两表查询，驱动表只会访问一次，被驱动表根据驱动表记录数访问多次。

   

#### 内连接和外连接

对于内连接的两个表，驱动表中的记录再被驱动表中找不到匹配的记录，该记录就不会加入到最后的结果集。

内连接语法：

SELECT * FROM t1 [INNER | CROSS] JOIN t2 [ON 连接 条件] [WHERE 普通过滤条件];

 

对于外连接的两个表，驱动表中的记录即使在被驱动表中没有匹配的记录，也任然需要加入到结果集。外链接又分为左连接（选取左侧为驱动表）和右连接。

 

对于where子句凡是不符合条件的都不加到结果集

对于on子句，外连接的话会把在被驱动表找不到的记录设置为null并加入到结果集，但放在内连接，其效果与where等效

 

基于块的嵌套循环连接：

当数据大的时候，我们不可能每次拿驱动表的一条记录挨个与被驱动表的记录对比，并且再拿下一条驱动表记录时候，又要把上次加载到内存又放回磁盘的被驱动表记录又要加载会内存，这要I/O代价特别大。

提出join buffer：执行连接查询申请固定大小内存，把驱动条记录部分或者全部装在到里面，然后与被驱动表比较，在内存中很快且减少了访问磁盘的次数

   

 

两表连接的成本：

连接查询总成本 = 单次访问驱动表的成本 + 驱动表扇出数 x 单 次访问被驱动表的成本

 

对于内连接需要优化两部分：

1、 尽量减少驱动表的扇出

2、 对被驱动表的访问成本尽量低

所以我们需要在被驱动表的连接列上建立索引，这样可以使用ref访问来降低成本，也可以在被驱动表上的连接列为主键或唯一二级索引，就是用const访问。

 

 

### Innodb统计数据存储方式

Innodb默认以表为单位收集和存储统计数据

CREATE TABLE 表名 (...) Engine=InnoDB, STATS_PERSISTENT = (1|0); 

ALTER TABLE 表名 Engine=InnoDB, STATS_PERSISTENT = (1|0);

STATS_PERSISTENT为1时存储到磁盘，0为内存。、

 

#### 定期更新统计数据

开启innodb_stats_auto_recalc。默认值是ON，不过自动更新数据时异步的

 

⼿动调⽤ANALYZE TABLE语句来更新统计信息

ANALYZE TABLE single_table；该方法为同步

 

innodb_stats_method决定着在统计某个索引列不重复值的 数量时如何对待NULL值。

 

### 事务

状态：

   

 

### Redo日志

记录对数据库的修改，在系统奔溃重启之后时按照redo日志步骤更新数据库。

好处：

1、 redo日志占用空间小

2、 redo日志是按顺序写入磁盘

 

 

#### 结构

   

简单的日志类型：

1、 MLOG_1BYTE（type字段对应的⼗进制数字为1）：表示在⻚ ⾯的某个偏移量处写⼊1个字节的redo⽇志类型。

2、 MLOG_2BYTE（type字段对应的⼗进制数字为2）：表示在⻚ ⾯的某个偏移量处写⼊2个字节的redo⽇志类型。同样还有4和8字节的

   

3、 MLOG_WRITE_STRING（type字段对应的⼗进制数字 为30）：表示在⻚⾯的某个偏移量处写⼊⼀串数据。

   

 

复杂的redo日志类型：

1、 MLOG_REC_INSERT（对应的⼗进制数字为9）：表示插⼊⼀ 条使⽤⾮紧凑⾏格式的记录时的redo⽇志类型。

2、 MLOG_COMP_REC_INSERT（对应的⼗进制数字为38）：表示 插⼊⼀条使⽤紧凑⾏格式的记录时的redo⽇志类型等等。

 

因为某些操作会修改多个页面而会产生很多个redo日志，mysql把他们分为一组，弄成原子性，使得他们不是全部生成就是全部没有生成日志。在每个组最后面加上一个type为MLOG_MULTI_REC_END的日志作为标记。已达到原子性。

 

Mysql会在type第一个bit特位来表示是多redo日志还是单一redo日志，如果为1则是产生一条redo日志。

 

 

#### MINI_TRANSCATION

对底层页面的一次原子访问的过程称为mrt。一个事务可以包含多条语句，一条语句可以包含多个mtr组成，一个mtr可以包含一组redo日志。

 

#### 写入过程

##### Redo日志页结构

   

##### Redo日志缓冲区

Redo日志不会直接写入磁盘，而是先写到log buffer缓冲区。

向log buffer中写入redo日志是顺序的，并且是按mtr的一组redo日志来插入，而不是一个redo日志就插入。

 

   

 

##### 日志刷盘

现在redo日志已经在缓存中，那么何时写入磁盘呢？

1、 log buffer空间不足

2、 事务提交

3、 后台自动刷新

4、 关闭服务器

5、 Checkpoint

 

Redo日志存到mysql下的data下的ib_logfile中，并且采用循环写入到方式。

 

 

##### Log Sequeue Number

简称为LSN。随着redo日志不断增加，该值也不断增加。规定初始的LSN值为8704!

每一组mtr生成的redo日志都要一个唯一的LSN值与其对应，LSN越小，该redo日志产生的越早。

 

 

##### Checkpoint

我们说redo日志是为了帮助之前没有成功更新数据的脏页是否刷新到磁盘，同时redo日志是循环写入的，那有个问题，如果我们准备想磁盘中redo日志覆盖一些，就要先判断覆盖的日志对应的脏页是否已经正常，否则不允许覆盖，

 

那么如何去看是否可以覆盖呢？

提出了一个叫checkPoint_lsn得值来确定当前可以被覆盖的redo日志总览。

对应的脏页被写入磁盘后，checkpoint_lsn就跟着增加。

   

 

##### 崩溃恢复

1、 确定起点：找到最近发生的checkpoint_lsn值

2、 确定终点：找到redo日志中某个不为512字节的页面，他就是终点

3、 恢复数据：1、会把所有都装到hash表中，根据相同spaceID和pageNuber的放到一个槽中，根据顺序用链表连在一起，这样的话可以方便对同一页面进行处理。2、跳过已经刷新的页面，已经我们获取的checkPoint_lsn可能不是最新的，比如刚跟新这个值，就有个脏页刷新到磁盘。所以之后的redo日志对应的脏页可能已经刷新了，所以我们需要查看每个页面的File HEADER中的FIL_PAGE_LSN，如果该页面已经刷新，则该值大鱼checkPoint_lsn。

 

   

 

 

### undo日志

为了回滚做准备而记录插入、更新、删除所需回滚的日志，叫undo日志

 

#### 事务id

对于只读会在它第一次对某个用户创建的临时表执行的时候才会分配

对于读写会在它第一次对某个表执行的时候才会分配。

 

跟row_id隐藏列一样，聚族索引记录还会保存一个叫trx_id的隐藏列和roll_pointer隐藏列

 

Roll_pointer就是指向某个undo日志的指针

 

   

 

#### 结构

插入所用的：

   

 

更新所用的：

 

更新主键和不更新主键的情况。

不更新主键：

1、 就地更新，也就是更新前后每个列所占用空间是一样的

2、 删除在插入：先真正把之前的记录删除，添加到垃圾链表中，插入新的记录

更新主键：

1、将就记录delete mask（在事务提交后才交由专门线程purge，加入垃圾链表

），根据更新后各列的值创建一条新纪录，将其插入到聚族索引中

 

 

 

   

 

 

 

#### undo的页结构

   

 

其中undo Page Header是其特有的，里面的属性如下：

其中type是该页面存什么类型的undo日志，undo日志分为两个insert和update两个，分开存放

   

一个事务有多个语句，每个语句对应多个记录个改动，每个记录改动对应1-2个undo日志，而且同一页面只能存放同一类型undo日志，所以一个事务可能有一下两个链表

   

并且规定了每一个undo页面链接对应一个段，称为undo log segement。那么每个链表头一个first undo page就有一个undo log segement header，里面存放了segement header，存放了基节点等信息

 

 

一个事务向一个undo页面添加undo日志的都归为一个组。每写入一组undo日志，就会存储该组属性，由Undo Log Header来负责。

 

   

 

 

 

 

### 事务隔离级别和MVCC

事务并发遇到的问题：

1、 脏写：一个事务修改了另一个事务修改但未提交的事务

2、 脏读：一个事务读到了另一个事务还未提交的事务

3、 不可重复读：一个事务第一次查询结果和第二次查询结果因为其他事务的修改二不一样

4、 幻读：一个事务第一次读的和第二次读，第二次数据变多了。

 

严重级别：脏写>脏读>不可重复读>幻读

 

隔离级别：

隔离级别             脏读      不可重复读     幻读 

READ UNCOMMITTED  Possible     Possible      Possible 

READ COMMITTED    Not Possible  Possible      Possible 

REPEATABLE READ    Not Possible  Not Possible   Possible 

SERIALIZABLE        Not Possible  Not Possible   Not Possible

 

 

 

#### MVCC

版本链

对于InnoDb存储引擎，表的聚族索引记录两个必要隐藏列

Trx_id:每次一个事务对某条聚族索引记录进行改动，都会把该事务id复制给该隐藏列

Roll_pointer:每次对某条聚族索引进行改动，都会把旧的版本写入到undo日志中，然后这个隐藏列就等同于指针，可以通过它找到该记录修改前的信息。

 

下面的两个事务对同一语句进行操作就会生出一个undo日志的版本链

   

 

 

 

 

版本链的头部记录了最新的值

   

 

##### ReadView

重要内容：

1、 m_ids:表示在生成readview时当前系统中活跃的读写事务的id列表

2、 min_trx_id:表示生成readview时当前系统中活跃的读写事务中最小的事务id，也就是m_ids最小值

3、 max_trx_id:表示生成readview系统中应该分配给下一个事务的id值，是累加的。注意他不是m_ids的最大值

4、 creator_trx_id:         表示生成readview的事务id

 

步骤：

1、 如果被人访问版本的trx_id与readview中的creator_trx_id相同，当前事务可以访问

2、 如果trx_id小于readview的min_trx_id，表明早已经提交，可以访问

3、 如果trx_id大于readview中的max_trx_id，表明还未提交，不可访问

4、 如果trx_id大于min_trx_id小于max_trx_id，则看他是否在m_ids列表中，在，则不可访问，不在就可以访问

 

###### 生成时机

readCommited和repeatable read的readview生成时机不一样

 

readCommited是在每次读取数据的时候生成一个readview

 

而repeatable read是在第一次读取数据时生成一个readview保证可重复读，另一个事务提交不影响结果

 

下面举个例子：

   

 

版本链就为

   

 

 

 

现在开始查询生成readview为m_ids[100,200],min_trx_id=100,max_trx_id=201,creator_trx_id=0

我们判断第一个trx_id=100，发现在m_ids列表中，那么不可访问，一次遍历直到访问到位80，发现他小于min_trx_id，可以访问，数据为刘备。

 

 

之后我们提交事务100，并在200中跟新数据。新的版本链就出现了

   

 

现在查询他。readview依然为m_ids[100,200],min_trx_id=100,max_trx_id=201,creator_trx_id=0

第一个为200，在列表中不可访问。然后访问到第三个事务id为100也不可访问，知道访问到为80，也就是数据为刘备。

 

我们可以看到前后两次数据是一样的，这也就是repeatable read消除的不可重复读。

 

 

 

 

### 锁

并发事务访问相同记录大致分为3段

1、 读-读情况：不需要加锁

2、 写-写情况：会发生脏写，

3、 读-写和写-读：发生脏读、不可重复读和幻读

 

#### 一致性读

事务利用MVCC进行的读取操作称为一致性读也称为快照读。所有普通Select语句在READ COMMITED和REPEATABLE READ隔离级别下是一致性读，是不加锁的。

 

#### 锁定读

共享锁：S锁，事务要读取一条记录，获取该记录锁

独占锁：X锁，事务要改动一条记录，获取该锁。

 

   

 

对读取记录加S锁：

Select 。。。 Lock In SHARE Mode

如果别的事务想获得X锁就被阻塞，直到S锁被释放

 

对读取记录加X锁：

SELECT … FOR update

 

 

#### 多粒度锁

意向锁：表级锁。

意向共享锁：IS锁，当事务在某条记录加S锁，为这个表加IS锁

意向独占锁：IX锁，当事务在某条记录加X锁，为这个表加IX锁

这个两个锁仅仅为了加表级别的锁时候快速判断表中的记录是否被上锁，避免用遍历方式来查看是否上锁。

 

#### Innodb锁

 

##### 行锁

也称为记录锁。

 

1、 record Locks：把一条记录锁上

2、 gap locks：间隙锁，为了解决幻读问题也就是插入数据问题。

3、 next-key Locks:锁住某条记录并且阻止其他事务在该记录之前插入数据

   

4、 隐式锁：一个事务对新插入的记录可以不显示加锁，但由于事务id存在，相当于加了一个隐式锁，别的事务在对这条记录加锁时候，会先帮助当前事务生成索结构，再添加自己的锁结构进入等待状态。

 

### 字符集

   

 

 

   

 

字符集和比较规则：

1、 列没有指定字符集和比较规则，则该列默认用表的字符集和比较规则

2、 表没有指定，则默认使用数据库的

3、 数据库没有指定，默认使用服务器的

 

 

 

客户端到服务器，服务器到客户端的编码转换。服务器根据character_set_client来进行解码客户端传过来的数据，根据character_set_conntection编码给具体的列使用，查询完后根据charcter_set_results对结果编码返回给客户端，客户端根据操作系统字符集解码。

   

可以使用set names 字符集  来代替下面三个

SET character_set_client = 字符集名; 

SET character_set_connection = 字符集名; 

SET character_set_results = 字符集名;

统一客户端和服务端的编码字符集。

 

## 间隙锁

间隙锁只在可重复读隔离级别下才有效

原则1：加锁的基本单位是next-key lock 是前开后闭区间

原则2：查找过程中访问到的对象才加锁

优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁

优化2：索引上的等值查询，向右查询时且最后一个值不满足等职条件的情况下，next-key lock退化为间隙锁

一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止

 

根据规则，再删除语句的时候尽量用limit，可以控制删除的条数，并且减少加锁范围

可重复度遵守二阶段锁协议，所有加锁的资源，都是在事务提交或者回滚的时候才释放。

其实next-key-lock就是由间隙锁和行锁组成的，是两个步骤
 
 

 

 

## 慢查询性能问题

\1.      索引没有设计好

\2.      Sql语句没写好

\3.      Mysql选错了索引

 

## Mysql恢复

Redolog是物理日志，记录数据的变化，并且该日志是循环日志，满了就重头开始

Binlog是逻辑日志，记录的是操作， 并且该日志满了后会重新生成新日志来存储。所以该日志起到归档作用。

 

 

通过binlog和redo log恢复数据

 

binlog写入流程

事务执行时候，把日志写到binlog cache中，事务完成后，把cache中日志写到binlog中。并同步到磁盘。清空缓存。

 

Redo log写入流程

生成log写入到mysql中的redo log buffer，在写入到磁盘的page cache中，最后持久化到磁盘。其中最后一步最耗时间。

 

binlog有三种格式statement、row和mixed

statement记录了真实的sql语句，并且可能造成主备不一致

row记录了需要操作的行和数据，很占空间

mixed格式：mysql自己判断sql语句会不会造成主备不一致，会就采用row不会就采用statement

 

 

 

## Wal 预写日志系统

得益于两个方面：

\1.      redo log和binlog都是顺序写，磁盘的顺序写高于随机写

\2.      组提交机制，可以大幅度降低磁盘的IOPS消耗

 

 

 

# 设计模式

需要新的设计方式，应对项目的扩展性，降低复杂度

\1.      分析项目的变化与不变部分，提取变化部分，抽象成接口+实现

\2.      那些功能是根据新需求来变化的

## 策略模式

分别封装行为接口，超类里放行为接口对象，在子类里具体设定行为对象。

原则：分离变化部分，封装接口。

## 单例模式

确保一个类最多只有一个实例，并提供一个全局访问点

 

## 组合模式

将对象聚合成树形结构来表现整体、部分的层次结构。

能以统一方式以一致的方式来处理个别对象以及对象组合。

也就是忽略对象组合和个体之间的差异

 

## 模板模式

方法的调用顺序以及规定好了，只需要根据自定义来实现某些或者全部方法

 

## 骨架抽象

该抽象类定义了需要子类去实现的方法和自己已经实现的方法，并且未实现的方法已经在实现的方法中有了明确的调用位置。

# Spring

## Bean

如果是采取默认的单实例的话，容器创建好之后就创建好bean了。

多实例情况下，容器创建好并不会创建对象，只有每次获取对象的时候才会创建对象

 

### @conditional

按照条件进行判断，满足条件给容器注册bean

### 给容器注册组件

\1.      包扫描+组件标注注解（@Controller，@Service等）

\2.      @Bean【方便导入第三方包里组件】

\3.      @Import【快速给容器导入组件】

\1.      还可以实现ImportSelector：返回需要导入组件的数组

\2.      接口ImportBeanDefinitionRegistrar：手动注册bean到容器中

​      

4.使用spring提供的FactoryBean

  默认获取到的是工厂bean的getObject获取到的对象，获取工厂bean本身需要在id前面加个&

 

### 容器bean生命周期

   

\1.      我们可以自动以初始化和销毁

@Bean中的init-method和destroy-method

\2.      通过让Bean实现InitializlingBean 和DisposableBean

\3.      使用Jsr250注解：@PostConstruct：在bean创建完成并且属性赋值完成，来执行初始化方法

 @PreDestroy：在容器销毁bean之前通知我们进行清理工作  

用在方法上

\4.      接口BeanPostProcessor bean的后置处理器

在bean初始化前后处理工作

postProcessorBeforeInitialization：在初始化之前

postProcessorAfterInitialization：在初始化之后

 

根据源码

在初始化之前会给bean属性赋值

this.populateBean(beanName, mbd, instanceWrapper)

之后才会初始化，先执行初始化之前->初始化->初始化之后

```
this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
this.invokeInitMethods(beanName, wrappedBean, mbd);
this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
```

 

 

spring底层不管是bean赋值、注入其他组件、@Autowire、生命周期注解等都是BeanPostProcessor来完成的

 

 

 

### 属性赋值

使用@Value

\1.      基本数值

\2.      可以spEl #{}

\3.      可以写${} 取出运行中环境变量

使用@PropertySource来获取配置文件

 

 

自动装配：利用依赖注入完成对IoC容器中组件的赋值

@AutoWired 默认优先按照类型去找对应组件

​            如果找到多个相同类型的组件，再将属性名称作为组件id去容器中查找

使用@Qualifier()可以明确指定组件id

利用@Primary 默认使用首选的bean

 

 

Spring还使用@Resource和@Inject，这两个都是java规范

@Resourece和@AutoWired自动装配，默认按照组件名称进行装配，不支持@Primary

@Inject要依赖inject包，没有required = false功能

 

 

自定义组件想要使用spring容器底层组件（ApplicationContext，BeanFactory）：

就要实现xxxAware，在创建对象的时候，会调用接口规定的方法来注入组件

XxxAware使用xxxProcessor来处理的

 

 

 

### @ Profile

Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能。

如开发环境、测试环境和生产环境

@ Profile指定组件在那个环境的情况下才能注册到容器中。

 

 

### AOP

\1.      需要将业务逻辑和切面类都加入到容器中，告诉Spring哪个是切面类  @Aspect

\2.      在切面类上的每一个通知方法上标注通知注解，告诉spring何时运行（切入点表达式）

\3.      开启基于注解的aop模式 @EnableAspectJAutoProxy

 

Aop原理：【看给容器中注册了什么组件？这个组件什么时候工作？组件工作的功能？】

\1.      @EnableAspectJAutoProxy 导入了

@Import({AspectJAutoProxyRegistrar.class})

他给容器中注册了一个AnnotationAwareAspectJAutoProxyCreator

2． AnnotationAwareAspectJAutoProxyCreator：实现了

SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware

BeanPostProcessor：会在bean创建前后完成任务

BeanFactorAware：会自动注入BeanFactory

 

流程：1.传入配置类，创建ioC容器

\2.      注册配置类，调用refresh（）刷新容器

\3.      RegisterBeanPostProcessor（beanFactory），注册bean的后置处理器方便拦截bean的创建

3.1．先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor

3.2.  给容器中加别的BeanPostProcessor

3.3.   优先注册实现了PriorityOrdered的BeanPostProcessor

3.4.   再注册实现了Ordered

3.5    最后注册没实现Ordered的

3.6    注册BeanPostProcessor，然后放到容器中

AnnotationAwareAspectJAutoProxyCreator创建成功

\4.      finishBeanFactoryInialization来完成beanFactory创建，创建剩下的单实例对象

每一个bean创建之前调用postProcessBeforeInstantitaion

\1.      判断当前bean是否在advisedBeans中（保存了所有需要增强的bean）

\2.      判断当前bean是否是基础类型的Advice、PointCut、Advisor等接口实现类或者是否切面

\5.      创建完成对象后

调用postProcessAfterInstantitaion

\1.      获取当前bean的所有通知方法 

\2.      保存当前bean到adviceBeans     

\3.      创建当前bean的代理对象（jdk或者cglib）

\4.      给容器返回增强了的代理对象

\6.     目标方法的执行

​         1.cgLibAopProxy.intercept()拦截目标方法的执行

​         2.根据ProxyFactory对象获取将要执行的目标方法拦截链

​                  1.保存所有拦截器

​                  2.遍历所有增强器并转换为Interceptor

​                  3.将增强器转为List<MethodIntercepto>

​                  拦截器链（每一个通知方法又被包装为方法拦截器，利用MethodInterceptor机制）

\3.      拦截器链触发过程CglibMethodInvacation的proceed方法

1.如果没有拦截器执行目标方法，或者拦截器的索引和拦截器数组一样大小就执行目标方法

2.链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完返回以后再来执行。保证了通知方法与目标方法执行顺讯

   

 

 

 

### 事务

Spring会对配置文件特殊处理，给容器中加组建的时候，多次调用方法只会从容器中获取组件。

[1.需要给方法加上@Transactional](mailto:1.需要给方法加上@Transactional)注解

```
2.需要加上@EnableTransactionManagement
```

来开启事务管理

```
3. 注册事务管理器
@Bean
public PlatformTransactionManager platformTransactionManager()throws Exception{
    return new DataSourceTransactionManager(dataSource());
}
```

 

 

 

原理：

1.@ EnableTransactionManagement导入了TransactionManagementConfigurationSelector

 

2.TransactionManagementConfigurationSelector有导入了AutoProxyRegistrar和ProxyTransactionManagementConfiguration

​         AutoProxyRegistrar：给容器注册一个InfrastructureAdvisorAutoProxyCreator组件；

​         他是一个后置处理器

​         InfrastructureAdvisorAutoProxyCreator：利用后置处理器在对象创建以后，包装对象，返回一个代理对象(增强器),代理对象执行方法利用拦截器链进行调用

​         ProxyTransactionManagementConfiguration：给容器注册了事务增强器（用来解析@Transacational注解的属性信息）和

注册事务拦截器（保存了事务属性信息和事务管理器）其底层是MethodInteceptor

 

### 扩展原理

#### BeanFactoryPostProcessor

beanFactory的后置处理器：在beanFactory标准初始化之后调用（标准化--所有bean的定义保存加载到benafactory，但没有bean的实例）

invokeBeanFactoryPostProcessors(beanFactory)；执行所有的BeanFactoryProcessor

如何找到他们？ 

\1.      直接在BeanFactory中找到所有类型是BeanFactoryPostProceesor的组件，并执行他们的方法

\2.      在初始化其他组件之前

 

#### BeanDefinitionRegistryPostProcessor 

extends BeanFactoryPostProcessor

在所有bean的**定义**信息将要被加载但是bean的实例还未创建，在beanFactortPostProcessor之前

beanFactory就是按照beanDefinition里存的定义信息来创建实例的

利用他可以给容器中额外添加组件

 

#### ApplicationListener

监听容器中发布的事件，事件驱动模型开发

监听ApplicationEvent及其子类

 

 

步骤：

\1.      写一个监听器来监听事件

2       把监听器加入到容器

\3.      只要容器中有事件发生，就能监听到 

 

原理：事件发布流程

\1.      publishEvent() 先发布事件

\2.      获取事件的多波器：getApplicationEventMulticaster()

\3.      获取所有的ApplicationListener

\4.      如果有Executor，可以支持使用Executor进行异步派发

\5.      否则，同步的直接执行listener方法 invokeListener(listener,evnet)

通过listener回调onApplicationEvent（）

 

 

@EventListener 原理是EventListenerMethodProcessor

其实现了SmartInitializingSingleton –》afterSingletonInstantiated（）

Ioc容器创建调用refresh（）

调用finishBeanFactoryInitialization（beanFactory）初始化剩下的单实例bean

 -》先创建单实例bean->getBean（）->获取所有单实例bean，判断是否是SmartInitializingSingleton，如果是就调用afterSingletonInstantiated（）

 

 

 

 

 

 

### 总流程

spring容器的refresh（）创建刷新

1、prepareRefresh（）刷新前的预处理 

1）、initPropertySources（）初始化属性，子类自定义处理

2）、getEnviorment（）.validateRequiredProperties()检验属性的合法

3）、earlyApplicationEvent保存容器早期的一些事件

2.obtainFreshBeanFactory（）获取beanFactory

1）、refreshBeanFactory（）创建beanFactory     

​             New DefaultListableBeanFactory()

2)、getBeanFactory（）返回创建的beanFactory对象

3）、将创建的对象返回

3.prepareBeanFactory（beanFactory）预准备工作

1）、设置beanFactory的类加载器、支持表达式解析器

2）、添加BeanPostProcessor【ApplicationContextAwareProcessor】

3）、设置忽略的自动装配接口EnviomentAware等

4）、注册可以解析的自动装配，我们能直接再任何组件中注入

​    BeanFactory、ResourceLoader、ApplicationPublisher、ApplicationContext

5）、添加BeanPostProcessor【ApplicationListenerDetector】

6）、添加编译时AspectJ

7）、给BeanFactory注册一些组件

4.postProcessBeanFactory；beanFactory准备工作完成后进行的后置处理工作

1）、子类通过该方法来对beanFactory创建并预处理完成以后做进一步设置

===============================以上是创建和预处理============================

5.invokeBeanFactoryPostProcessor

先执行BeanDefinitionRegistryPostProcessor在执行BeanFactoryPostProcessor

\6.      registryBeanPostProcessor  注册beanPostProcessor

不同类型的BeanPostProcessor在bean创建前后执行时机是不一样的

BeanPostProcessor

DestructionBeanPostProcessor

InstantiationAwareBeanPostProcessor

SmartInstantiationAwareBeanPostProcessor

MergedBeanDefinitionPostProcessor

1）、获取所有的BeanPostProcessor

7.initMessageSource（）国际化和消息绑定、解析

8.initApplicationEventMulticaster（）初始化事件派发器

9.onRefresh()留给子类在容器刷新的时候可以自定义逻辑

10.registerListeners（）将所有的ApplicationListener注册到容器中来

11.finishBeanFactoryInitialization（）初始化所有的单实例bean

12.finishRefresh（）完成BeanFactory的初始化创建工作，IOC容器创建完成

 

### 总结

1）、spring容器在启动的时候，会先保存所有注册进来的bean的定义信息

​         1）、xml注册bean

​         2）、注解注册bean

2）、当容器中有保存这些bean的定义信息时候，会在合适的时机创建bean

​         1）、用到这个bean的时候；利用getBean方法创建，创建好以后保存到容器中

​         2）、统一创建剩下的所有单例bean时候

3）、后置处理器

1）、每一个bean创建完成都会使用各种后置处理器进行处理，来增强bean的功能

AutoWiredAnnotationBeanPostProcessor：处理自动注入

AnnotationAwareAspectJAutoProxyCreator：处理Aop

4）、事件驱动模型：

​         AppliicationLisitener：事件监听

​         ApplicationEventMultiCaster：事件派发

 

# SpringMvc

## 流程

1、 加载:web.xml中获取dispatcherServlet、并且获取其init-param参数中配置的配置文件、再配置url-patten需要拦截的url

2、 初始化：init（初始化9大组件）、扫描指定包下的类和注解（@Controller等）、实例化交给spring容器管理、handlerMapping映射对应url和方法

3、 运行：doDispatcher拦截符合的请求(从handlerMapping里查到对应的方法)、反射调用method、返回结果

 

## 组成

Springmvc的servlet有三层：

1、 HttpServletBean:直接继承自java HttpServlet，将Servlet中配置的参数设置到相应属性中

2、 FrameWorkServlet初始化了WebApplicaitonContext

3、 DispatcherServlet初始化自身9大组件

 

### 9大组件

HanderMapping、HandlerApater和ViewRosvler最主要

 

 

# Tomcat

## 共享库和运行时插件

\1.      servlet容器启动会扫描当前应用每一个jar包的ServletContainerInitializer的实现

\2.      提供其实现类，必须绑定在META-INF./services/javax.servlet.ServletContainerInitializer,在里面写上实现类的全类名

应用启动的时候运行他的onStartUp方法，配合@HandlersTypes注解传入感兴趣的类和其子类，通过Set<Class<>> arg可以获取它们。

可以通过servletContext来注册组件，通过添加组件来获取一个动态的类，并且配置映射信息

 

```
        //注册组件
FilterRegistration.Dynamic filter = servletContext.addFilter("gameFilter",GameFilter.class);
ServletRegistration.Dynamic dynamic1 = servletContext.addServlet("helloServlet",HelloServlet.class);
//配置映射
filter.addMappingForServletNames(EnumSet.of(DispatcherType.REQUEST),true,"helloServlet");
dynamic1.addMapping("/hello");
```

 

 

#### springmvc

springmvc采用了上面的方式

\1.      Web容器在启动的时候会扫描每个jar包下的META-INF./services/javax.servlet.ServletContainerInitializerJava

\2.      加载这个文件指定的类SpringServletContainerInitializer

\3.      Spring的应用已启动就会加载WebApplicationInitializer接口下的所有组件

\4.      并且为WebApplicationInitializer组件创建对象

1）、AbstractContextLoaderInitializer：创建根容器，createRootApplicationContext（）

2）、AbstractDispatcherServletInitializer：

​         创建一个web的IOC容器，createServletApplicationContext（）

​         创建DispatcherSerlvet：createDispatcherServlet（）

​         将创建的DispatchServlet添加到ServletContext中

3）、AbstractAnnonationConfigDispatcherServletInitializer：

​         创建根容器和web的ioc容器

 

总结：已注解的方式启动springmvc，继承AbstractAnnonationConfigDispatcherServletInitializer

 

 

## 模型

Tomcat的两个核心功能：

\1.      处理socket连接，负责网络字节流与Request和Response的对象转换

\2.      加载和管理Servlet，以及处理Request请求

 

Tomcat设计了两个核心组件连接器和容器，连接器负责对外交流，容器负责内部处理。

 

### 连接器

连接器需要完成的三个功能

\1.      网络通信

\2.      应用层协议解析

\3.      Tomcat request/response 与 ServletRequest/ServletResponse的转换

Tomcat设计了三个组件来实现这三个功能：EndPoint、Processor、Adapter

EndPoint负责提供字节流给Processor，Processor负责提供Tomcat Request给Adapter，转换为ServletRequest给容器

 

由于Io模型和应用层协议的互相组合。Tomcat设计了一个ProtocolHandler来封装endPoint和Processor

   

 

#### NioEndPoint （I/O多路复用）

   

流程：LimitLatch来管理获得连接数量，acceptor来监听连接，当有连接时候，accptor通过accept方法来返回Channel对象给Poller，Poller内部维护Channel数组，当发现一个Channel可用的时候，就会返回一个SocketProcessor（实现了Runnerble接口）给Executor去执行他的run方法，并且调用其Http11Processor来读取和解析数据

 

#### Nio2EndPoint（异步I/O）

   

可以看到该模型就没用了Poller（Selector），那是因为异步I/O的情况下，Selector的作用没代替了。

 

#### APREndPoint

##### APR

Apache可移值运行时库，其目的地是向上层应用程序提供一个跨平台的操作系统接口库。Tomcat用他来处理文件和网络I/O。

 

APREndPoint通过JNI调用APR库来实现非阻塞I/O

   

可以看到APREndPoint和NioEndPoint流程几乎一样，差异在于Acceptor和Poller使用的APR库的操作系统指令。

 

##### APR提升性能

除了APR自身库的原因外还有以下原因：

1.JVM堆  vs  本地内存

   

 

HeapByteBuffer和DirectByteBuffer区别

HeapByteBuffer对象本身在堆内存上，并且持有的数组byte也是在堆上，如果其接收数据时候，需要把数据从内存拷贝到本地内存，再从本地内存拷贝到堆上。为什么不能直接把其从内存上拷贝到堆上呢。因为直接从内存到堆，可能jvm会发生GC，可能就会造成原先的byte数组地址变化那么拷贝就会失败。而jvm可以保证从本地内存到堆上的拷贝不会出现GC。

DirectByteBuffer与其不同的是，它的byte数组分配不是在堆上，而是在本地内存，其记录了他在本地内存的地址。

 

###### jvm延伸

那么为什么jvm可以保证本地内存到堆上的拷贝不会出现GC呢

这是HotSpot VM层面保证的，具体来说就是HotSpot不是什么时候都能GC的，需要JVM中各个线程都处在“safepoint”时才能GC， 本地内存到JVM 堆的拷贝过程中是没有“safepoint”的，所以不会GC。关于这部分内容在深入理解java虚拟机中43页可以看到相关解释。

 

 

2.通过操作系统的sendfile避免了内核与应用程序之间的内存拷贝以及用户态和内核态的转换。

### 容器

Tomcat有四种容器。Engine表示引擎，用来管理多个虚拟站点，一个service最多只能有一个engine，host代表一个虚拟站点，可以给tomcat配置多个虚拟站点，一个虚拟站点对应多个web应用程序，context表示一个web应用程序，warpper表示一个servlet，一个web应用程序有多个servlet

   

 

   

 

 

Tomcat利用组合模式管理这些容器。具体实现方法，所有容器实现了Container接口。采用了Pipeline-Valve责任链模式如下图

   

Valve和Filter区别：

1． Valve是tomcat私有的，与tomcat耦合，servletAPI是公有标准，所有web容器都支持Filter

2． Valve是web容器级别，filter是应用程序级别

 

 

### 层次架构

   

 

Tomcat组件的功能是由Container接口来规定的，而生命周期就是交给了LifeCycle接口。体现了**接口分离**

 

### 启动tomcat

在使用脚本startup.bat启动tomcat流程如下

   

Catalina主要任务创建server，他需要先解析serve.xml文件，并把文件配置的组件都创建出来，最后调用server组件的init（）和start（）方法

 

 

### tomcat线程池

可以参考java线程池

Tomcat线程池扩展了java原生的ThreadPoolExecutor，通过重写exeute方法实现了自己的任务处理逻辑：

\1.      前corePoolSize，来一个任务就创建一个线程

\2.      再来任务的话，就会把任务放到工作队列让所有线程去抢，如果队列满了就创建临时线程。

\3.      如果总线程数大道maximumPoolSize，则继续尝试把任务添加到队列

\4.      如果添加不成功，插入失败，执行拒绝策略

### WebSocket

webSocket工作原理：通过http协议握手后，使用tcp的socket来进行传输。并且数据已帧的形式按照先后顺序进行传输。

 

 

tomcat中的websocket加载是通过ServletContainerInitializer来实现的。里面指定了扫描一个叫ServerEndPoint和EndPoint类及其子类，并且创建容器来维护一个包含url和对应的EndPoint映射关系。

 

## 扩展

### Jetty线程策略

Selector一般思路：启动一个线程，该线程通过select()不断检查状态，如果状态改变就把I/O事件包装为一个Runnable。然后把Runnable给另外一个线程进行处理。

然而这个好处在于解耦并且一般情况下效率不错，但是在会造成CPU缓存无法使用，并且线程切换浪费资源。

所以Jetty把I/O事件的检测和处理放到同一个线程，在不阻塞的情况下，可以充分使用CPU缓存。他就是ManagedSelector

 

#### ManagedSelector

这个类里面最关键的就是update队列和执行策略

 

在ManagedSelector里对Selector状态进行操作被封装到了一个SelectorUpdate接口里面，而不能直接调用Selector去更改。而对应的也就是I/O事件变化。

那么当I/O事件变化时候，谁来执行呢？

需要某个组件实现Seletable接口的onSelected方法，他会返回个Runnable，ManagedSeleor拿到后交给线程池来处理。

 

ManagedSelector如何统一管理和维护用户注册的Channel集合？

通过ExecutionStrategy接口，任务生成委托给内部接口，而具体执行策略由当前线程还是新线程有具体实现逻辑再决定。

 

jetty提供了ProduceConsume、ProduceExecuteConsume、ExecuteProduceConsume、EatWhatYouKill

ProduceConsume：就是依次生成和任务执行

ProduceExecuteConsume：生产和执行是不同线程

ExecuteProduceConsume：生产者自己执行任务。但是如果遇到I/O事件执行太长，会导致大量线程阻塞和线程饥饿。

EatWhatYouKill：他会在线程少的时候执行ExecuteProduceConsume，线程比较多的时候采用ProduceExecuteConsume。可以说他是一种平衡。

 

### 对象池

tomcat和Jetty因为每次连接都要创建大量复杂对象，而连接一般处理也很短，就会造成大量开销。他们两个都采用对象池技术来解决这个问题。

 

tomcat采用SynchronizedStack来作为对象池，底层是栈，并且支持扩容。

为什么tomcat不用链表而用数组，因为数组维护起来简单，并且支持扩容不支持缩容。

 

思考：

1、 使用对象池会产生同步问题就会需要额外的锁来处理。所以在设计上尽量减少使用锁。如果内存足够大可以采用ThreadLocal来保证线程之间不干扰。

2、 池化技术可能会有内存泄漏。池化本质是集合对象，集合对象不被GC，那么里面存储的缓存对象就不会被GC，并且维护大量对象会占用内存，需要提供主动的清理对象方法。

 

 

 

# Java基础

## 基本类型

   

 

 

*java中整数类型默认的int类型；小数类型默认的double；

*char 可以当做一中特殊的整数类型；

*int无法转换为boolean；

*小数类型转为整数类型，小数可能被舍弃，所有出现精度损失，所以需要强制转换；

*boolean 类型不能转换成任何其它数据类型；

 

   

 

## 通信模型

对于网络I/O通信过程，比如网络数据读取，会涉及到两个对象，一个是调用这个I/O操作的用户线程，另一个就是操作系统内核。一个进程的地址空间分为内核空间和用户空间，用户线程不能直接访问内核空间。

所以操作一般分为两步：

\1.      用户线程等待内核将数据拷贝到用户空间

\2.      内核线程将数据从内核空间拷贝到用户空间

 

阻塞/非阻塞：应用程序在发起I/O时候，是立即返回还是等待。

同步/异步：是指应用程序在和内核通信时候，数据从内核空间到应用空间的拷贝，是内核主动发起的还是应用程序调用的。

 

 

## 集合

### Arrays.asList

   

 

如何转换？

\2. 最简便的方法(推荐)

List list = new ArrayList<>(Arrays.asList("a", "b", "c"))

\3. 使用 Java8 的Stream(推荐)

Integer [] myArray = { 1, 2, 3 };
 List myList = Arrays.stream(myArray).collect(Collectors.toList());
 //基本类型也可以实现转换（依赖boxed的装箱操作）
 int [] myArray2 = { 1, 2, 3 };
 List myList = Arrays.stream(myArray2).boxed().collect(Collectors.toList());

\5. 使用 Apache Commons Collections

List<String> list = new ArrayList<String>();
 CollectionUtils.addAll(list, str);

 

 

 

## 线程池

#### 线程状态

   

#### Executor

Executor框架将任务的提交与执行解耦开来。并用Runnable表示任务。Excutor基于生产者-消费者模型。提交任务的操作相当于生产者，执行任务的线程相当于消费者。

 

##### 生命周期

为了解决Executor的生命周期问题，executorService扩展了executor，添加了生命周期管理的功能。

 

状态：运行、关闭和已终止。其中shutdown方法执行平缓关闭，不再接受新任务，等待已经提交任务的执行完成，而shutdownNow会粗暴的取消所有正在执行的任务。

 

##### Runnable缺陷

Runnable这种任务局限性在于不能够返回结果或者抛出异常给与我们信息。下面我们就引出Callable接口来返回结果数据。当然光有这些还不够，我们有提供了一个Future接口来异步的获取计算状态和结果。

 

 

##### 获取多结果

我们可以通过轮询多个Future来获取多计算结果，但消耗特别大且繁琐。

 

这里提供了CompletionService接口，该接口提供了take和poll方法返回Future，而这个是专门为BlockingQueue准备的。

 

ExecutorCompletionService实现了该接口。他的原理是在通过内部的BlockingQueue来保存Future最后的结果。如何保存的呢？是在内部QueueingFuture实现了FutureTask接口（该接口继承了Future和Runnable），并且重写了future的done方法，会在完成时候把Future添加到阻塞队列里面。之后通过take和poll来获取结果

 

 

#### 线程池创建

通过Executors的静态工厂方法来创建一个线程池：

newFixedThreadPool是一个固定长度的线程池

newCachedThreadPool创建一个可缓存的线程池，如果当前规模超过了处理需求，就会回收空闲的线程，当需求增加时，添加线程，没有大小限制。

newSingleThreadExecutor是一个单线程的Executor。

newScgedykedThreadPool创建一个固定长度的线程池，而且以延迟或定时来执行任务。

 

 

 

#### 扩展ThreadPoolExecutor

提供给子类beforeExecute、afterExecute和terminated。这也是一种骨架抽象(提供给子类实现的方法)和模板方法(方法的调用步骤以及规定好了)的结合。

 

## Nio

三大核心组件：Selector、Channel、Buffer

 

Channel和Buffer关系

   

 

Selector允许单线程处理多个Channel

   

### Channel

可以称为流下面是他的一些特性：

1、 既可以从通道读数据，也可以向通道写数据

2、 通道可以异步读写

3、 通道中的数据总是要先读到Buffer或者从一个Buffer写入

 

这些是Java NIO中最重要的通道的实现：

·        FileChannel

·        DatagramChannel

·        SocketChannel

·        ServerSocketChannel

FileChannel 从文件中读写数据。

DatagramChannel 能通过UDP读写网络中的数据。

SocketChannel 能通过TCP读写网络中的数据。

ServerSocketChannel可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。

 

Channel也可以在互相传输

 

 

### Buffer

#### 流程

使用buffer读取数据需要遵守下面步骤

1、 写入数据到buffer

2、 调用flip（）

3、 从buffer中读数据、

4、 调用clear（）

flip（）会将buffer模式转换，从写模式切换为读模式

 

#### 原理

三个重要属性：capacity、position、limit

capacity：表示内存大小

position：写模式下表示当前写入位置，读模式下，position置为0

limit：写模式下为capacity，读模式下为position写模式的大小

 

   

 

buffer中rewind（）能在读模式下把position又置为0，重新读取数据

 

### Selector

​         Selector起到监控与管理作用。

 

与Selector一起使用时，Channel必须处于非阻塞模式下。这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式。而套接字通道都可以。

 

先是启动Selector，并且与channel绑定，获得SelectionKey，持续的从中获得获取各个通道的key，并且判断key中的事件。

 

示例代码：

Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);

while(true) {

  int readyChannels = selector.select();

  if(readyChannels == 0) continue;

  Set selectedKeys = selector.selectedKeys();

  Iterator keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

​    SelectionKey key = keyIterator.next();

​    if(key.isAcceptable()) {

​        // a connection was accepted by a ServerSocketChannel.

​    } else if (key.isConnectable()) {

​        // a connection was established with a remote server.

​    } else if (key.isReadable()) {

​        // a channel is ready for reading

​    } else if (key.isWritable()) {

​        // a channel is ready for writing

​    }

​    keyIterator.remove();

  }

}

 

 

### Pipe

pipe是2个线程单方向的传输数据。通过sink和source两个通道。

   

 

## JUC

Jdk1.5以后提供了这个并发包

 

Volatile：当多个线程进行操作共享数据时候，可以保证内存中数据可见性

Volatile和synchronized

1、 volatile不具备互斥性

2、 不能保证变量原子性（i++就不是原子性，他是自增和赋值两个原子操作）

 

 

### 原子变量

那么出现i++这种情况怎么办？

使用JUC提供的原子变量来解决AtomicInteger等

他是如何解决的？

1、 使用了volatile保证内存可见性

2、 使用CAS（Compare-and-Swap）算法保证了数据的原子性

是硬件对于并发操作共享数据的支持

 

### 容器

容器的toString（）、hashCode（）、equals（）等方法都会间接触发容器的迭代。可能会抛出CocurrentModificationException

 

 

通过java5.0提供的并发容器来代替同步容器在多线程的情况下并发的不足。

 

阻塞队列适用于生产者-消费者模型。Deque双端队列适用于工作密取。

什么是工作密取？每个消费者都有自己的双端队列，如果一个消费者完成了自己双端队列的任务，那么他可以从其他消费者双端队列末尾秘密获取工作来完成。

有更好的伸缩性和有效降低了竞争关系。

 

ConcurrentHashMap

而HashTable是一个锁给锁住了，等于是串行，效率很慢。

默认分成16个段，每个段都是独立的锁。

   

 

但是在JDK1.8后，这个类取消了分段锁。而采用CAS来实现。

 

CopyOnWriteArrayList：添加操作过多不应该采用它。因为他底层是每次添加就要复制到一个新的数组上面去。

 

 

#### 同步工具类

可以使任意一个对象，只要他根据其自身的状态来协调线程的控制流。

 

 

CountdownLatch（闭锁）一个同步辅助类，在完成一组正在其他线程中执行的操作之前，允许一个或多个线程一直等待。

1、 确保某个计算再起需要的所有资源都被初始化之后才继续执行。

2、 确保某个服务再起依赖的所有其他服务都已经启动后才启动

3、 等待某个操作所有者都准备就绪再继续执行

​                                                                                                                                                                                                                                                                                                                                                             

Futuretask也可以用作闭锁。Future的get方法的行为去取决于任务的状态。如果任务完成，那么get立即返回结果，否则将阻塞直到任务进入完成状态或者抛出异常。FutureTask将计算结果从执行计算的线程传递到获取这个结果的线程。确保了传递过程能实现结果的安全发布。

 

Semaphore（信号量）。计数信号量用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。可以用来实现某种资源池。或者容器添加边界。

Semaphore管理者一组虚拟许可。执行操作时需要先获取许可，并在使用以后释放许可。如果没有许可acquire方法将阻塞到有许可为止。

 

 

 

## 多线程

### ThreadLocal

分为三个重要对象Thread、ThreadLocal、ThreadLocalMap

Thread对象保存了ThreadLocalMap，每个线程对应一个

ThreadLocalMap的每个Entry存储了k-v  = ThreadLocal对象-实际存储的值

ThreadLocal来设置和获取当前下线程的value值

 

   

 

**如何实现隔离？**

每个线程对应一个ThreadLocalMap

 

**对象存储和获取原理？**

设置的key为ThredLocal对象，value为存储的值，会获取到当前下线程的ThreadLocalMap对象并存储在里面

获取方法也是先获取当前线程ThreadLocalMap对象，之后根据该对象获取到对应ThreadLocal的值

 

**内存泄露问题**

因为ThreadLocalMap的key值为弱引用，所以会出现在下一次GC的时候key为null，而value还存在永远无法访问的情况。

 

设计者已经为其提供了自动清理，但是有的时候需要手动remove清理。

可以把threadLocal设置为static强引用。就不会出现之前的情况了。

 

**为什么要用弱引用？**

如果使用强引用，就会使得其生命周期与ThradLocalMap一样长，就不会被GC，如果没有手动删除，就会导致Entry内存泄漏。

 

**ThreadLocalMap****如何解决冲突？**

开放地址法

 

 

 

 

 

 

# JVM

## java内存区域

### 私有内存

#### 程序计数器

是当前线程所执行的字节码的行号指示器

 

#### java虚拟机栈

是java方法执行的内存模型。在方法执行时候会分配一个栈帧来存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法的调用到完成对应一个栈帧在虚拟机栈中入栈到出栈的过程。

 

其中局部变量表存储了基本数据类型和对象引用。并且除了long和double各占2个局部变量空间Slot意外，其余各占一个。

 

局部变量表在编译期间就完成分配，大小确定并且不会改变。

 

#### 本地方法栈

本地方法栈使用的时Native方法服务

### 公有内存

#### java堆

几乎所有对象的实例都在这里被分配。是内存中最大的区域。

并且也是垃圾收集器管理的主要区域。java堆分为新生代和老年代也有的叫Eden、From Survivor、To Survivor。

 

#### 方法区

逻辑上属于堆。

他存储了类的方法代码、变量名、方法名、访问权限、返回值等。

 

#### 运行时常量池

属于方法区

用于存储class文件的符号引用（全限定名如com.thoughtworks.Tool）和字面量（值）

 

因为是运行时所以是动态的

运行期间也可以把常量放入池中

用的最多的String.intern()à查看常量池是否存在该字符串，有就返回，没有的话就从堆中拿出该对象引用放入到字符串常量池（在jdk1.7之后，字符串常量重新被移到了堆中。）

 

字符串对象存储问题：

\1.      直接定义字符串变量的时候赋值，如果表达式右边是字符串变量存在常量池

\2.      new出来的放到堆中

\3.      对字符串拼接如果右边是纯字符串则放到常量池，如果不是放到堆中

 

 

**Jdk1.8****更新以前方法区属于堆内存的永久区，后面修改后去掉了永久区。方法区移到叫元空间的地方，使用了物理内存对应就是电脑内存** 

 

### 对象内存布局

对象在内存中分为三部分对象头、实例数据和对齐填充。

1、 对象头包括两部分

第一部分：用于存储对象自身运行时数据，如哈希码、GC分带年龄、锁状态标志等。官方称为Mark Word。并且因为节省空间的原因，Mark Word会根据当前对象状态来复用存储空间，下面是不同状态下MarkWord存储内容

第二部分：存储类型指针，虚拟机通过该指针来确定该对象属于哪个类

2、 实例数据：对象真正存储的有效信息，是代码中定义的的各种类型的字段信息。

3、 对齐填充：仅仅起占位作用。为什么有他？因为HotSpot Vm自动内存管理器要求对象起始地址必须是8字节的整倍数，也就是对象大小必须是8字节的整数倍，因为对象头是满足要求，所以一般是实例数据不是8整数倍时候才需要对齐填充。

 

   

 

### 对象的访问地址

有两种访问方式：句柄访问和直接指针访问

句柄访问：在java堆中划出一块内存作为句柄池，reference存的是对象句柄地址

 

 

   

直接指针：reference存的是对象地址

   

 

区别：句柄访问是把实例数据和类型数据分开存储。并且当出现GC等情况的时候，reference无需修改，只需要改变句柄池中指向地址。但是每次访问都需要多定位一次。

​         直接指针把实例数据和类型数据放在一起。每次访问都很快。但是当出现GC时候，都需要对reference改动。

​         虚拟机Sun HotSpot采用的是直接指针访问对象。    

 

## 垃圾收集器和内存分配策略

在堆和方法区中在编译期，对象是不确定的。只有在运行时候才能知晓，所以他们的内存分配和回收是动态的。

 

### 对象存亡？

通常采用可达性分析来判断对象是否不可用。

基本思想：通过一些称为GC ROOTS的对象作为起始点向下搜索，搜索所走过的路径称为引用连，当GC ROOTs不可达某对象时候，该对象就不可用了。

 

在java中可称为GC ROOTS对象的：

1、 虚拟机栈（栈帧中的本地变量表）所引用的对象

2、 方法区中静态属性引用的对象

3、 方法区中常量引用的对象

4、 本地方法栈中所引用的对象

 

 

### 引用

Jdk1.2以后对引用扩充如下

\1.      强引用：类似Object obj = new Object()只要强引用还在，垃圾收集器永远不会回收掉被引用对象

 

\2.      软引用：是还有用但并非必须对象。软引用关联的对象在系发生内存溢出前，第一次gc不会扫描他，第二次会包括。相当于缓存

 

\3.      弱引用：是非必须对象。垃圾收集器工作时候，无论当前内存是否足够，都会回收掉被弱引用关联的对象。

为什么需要弱引用：如果有一个值，对应的键不再使用，将会出现什么情况？假定对某个键的引用已经消亡，不在有任何途径引用这个值的对象对应的这个键/值对还无法删除，垃圾回收无法删除他。

为什么？因为映射对象是活动的就无法回收，解决办法就是通过设置键为WeakReference就可以删除对应条目。

 

在java源代码中ThreadLocal里面的ThreadLocalMap的键值Entry就是WeakRefence

 

\4.      虚引用：虚引用无法取得对象实例，设置其的目的是在这个对象被收集器回收时候收到一个通知。

### 垃圾收集算法

#### 标记-清除算法

算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象，之所以说它是最基本的收集算法，是因为后续的收集算法都是基于这种思路并对其不足进行改进而得到的。

它的主要不足有两个：

　　一是效率问题，标记和清除效率都不高，二是空间问题，标记清除后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后程序在运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作

 

   

 

​                                                                                    

#### 复制算法

为了解决效率问题，一种称为“复制”（Copying）的收集算法出现了，他将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这块的内存用完了，就将还存活这的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。只是这种算法的代价是将内存缩小为了原来的一半，未免太高了一点。

   

#### 标记-整理算法  

对于“标记-整理”算法，标记过程仍与“标记-清除”算法一样，但是后续步骤不是直接对可回收对象进行清理，而是让所有的存活对象都向一端移动，然后直接清理掉端边界以外的内存

   

 

#### 分代收集算法

根据各个生命周期在内存中划分不同的区域选择相应的收集算法来回收

 

#### GC的实现问题

因为可达性分析是根据GC ROOTS来搜索的，而GC ROOTS节点主要有全局性引用（常量、静态属性）与执行上下文（栈帧中的本地变量表）。虚拟机不会挨个检查引用因此需要知道对象引用位置，HotSpot引出了OopMap数据结构，会在特定位置指出哪些位置是引用。

上面所提到的特定位置也称为安全点safePoint，当线程运行到这类位置时候，堆对象状态是一致的，jvm可以安全的GC

 

那么安全点一般在什么位置？

\1.      方法返回之前

\2.      调用某个方法之后

\3.      抛出异常的位置

\4.      循环的位置

 

为什么选用这些点？

避免程序长时间无法进入safePoint，       

 

 

#### 内存和回收策略

1.6以前，方法区由永久代实现

   

 

Jdk1.8以后，为了把hotspot和jrokcit合并，永久代改为了元空间。

 

Minor GC：发生在新生代的垃圾收集动作，非常频繁，回收速度快

Major GC/Full GC：发送在老年代的GC

 

对象优先在新生代Eden区，当Eden区没有足够内存时候，虚拟机进行Minor GC。

Minor GC刚开始发生时候，对象只存在于Eden区和叫from的suvivor区，Eden区剩余的对象移到叫to的Suvivor区，from中的对象根据年龄值决定去向，（）可由-XX:MaxTenuringThreshold参数设置）到老年代，然后from和to空间交换位置

 

 

大对象进入老年代，所谓大对象指大量连续内存空间的Java对象，最典型的字符串以及数组。

长期存活的对象进入老年代。对象在Eden经过Minor Gc后，被Survivor容纳，当经过多个Minor GC后就会进入老年代。

 

空间分担担保：因为新生代只用一个Survivor进行备用，如果Eden时候活下来的对象太多，并且Survivor空间不够，就需要老年代的空间来帮助容纳。                

 

老年代一般标记清除或者标记清除与标记整理的混合实现

 

## 类文件结构

为了实现平台无关性和语言无关性，出现了一种中间语言—字节码

 

### 类文件结构

Class文件是一组以8位字节为基础单位的二进制流。它分为无符号数和表两种数据类型。

 

无符号数是基本数据类型，以u1，u2，u4，u8分别表示1,2,4,8字节的无符号数。可以用来表示数字、索引应用、数量值或者按照UTF-8编码构成的字符串值。

 

表是由多个无符号数或者其他表作为数据项构成的复合数据类型。

 

整个class文件本质就是表。

​          

 

下面我们来一一介绍

#### 魔数与java版本

每一个class文件头四个字节被称为魔数。唯一作用确定这个文件是否为一个能被虚拟机接受的class文件。

 

紧接着魔数的4个字节是class文件的版本号，第5,6字节是次版本号，7,8字节是主版本号。

 

#### 常量池

主次版本号之后就是常量池入口。因为常量池大小不定，所以在他之前放置另一个u2大小的计数器存储常量池大小。

 

需要注意的是常量池容量计数是从1开始而不是0，比较特殊！

 

常量池主要存储字面量和符号引用。

字面量是文本字符串、声明为final的常量值等

符号引用包括:

1、 类和接口的全限定名

2、 字段的名称和描述符

3、 方法的名称和描述符

 

 

#### 访问标志

紧接着两个字节是访问标志，识别一些类或者接口层次的访问信息。如这个Class是类还是接口，是否定义为public等等

 

#### 类索引、父类索引与接口索引结合

类索引和父类索引都是一个u2类型数据

而接口索引结合是一组u2类型数据。并且它的前两个字节是用来作为接口计数器存储容量。

 

#### 字段表集合

字段表集合用来描述接口或者类中声明的变量。

 

包括类级别变量和实例级别变量但不包括方法中声明的局部变量。

#### 方法表集合

与字段表集合类似。方法的代码的字节码指令存放在方法表集合下的属性表集合中一个叫Code的属性里面。

#### 属性表集合

属性表中比较重要的属性就是Code属性。

如果把java程序中信息分为代码（Code）和元数据（metaData）两部分，那么在整个class文件中，Code属性用来描述代码，所有其他数据项目用来描述元数据。

 

 

#### 虚拟机指令

加载和存储指令用于将数据在栈帧中的局部变量表和操作数栈之间来回传输

1、 将一个局部变量加载到操作数栈：load

2、 将一个数值从操作数栈存储到局部变量表：store

3、 将一个常量加载到操作数栈：const

## 类加载机制

###   概述

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化最终形成可被虚拟机使用的java类型，叫做类加载机制。

 

下面是类的整个生命周期。

   

值得注意的是解析可以能是在初始化之后开始。如People p = new BlackPeople（）这种会在使用时才会把p指向时机对象内存。

 

对于初始化阶段，虚拟机规定有且只有5种情况必须立即对类进行初始化：

1、 使用new创建对象、读取或者设置一个类的静态字段

2、 对类进行反射调用

3、 初始化一个类时候，如果父类没有初始化，先触发父类初始化

4、 当虚拟机启动时候，用户需要制定一个主类，先初始化这个类

5、 JDK1.7动态语言支持，如果MethodHandle解析结果为REF_getStatic等方法句柄，这个句柄对应类没有初始化，初始化这个类。

 

这五种场景称为主动引用。除此之外的都叫被动引用。

 

### 生命周期

#### 加载

加载需要完成：

1、 通过一个类的全限定名来获取定义此类的二进制字节流

2、 将这个字节流所代表的静态存储结构转换为方法区的运行时数据结构

3、 在内存中生成一个代表这个类的java.lang.class对象，作为方法区这个类的各种数据访问入口

##### 类加载器

虚拟机把实现加载阶段的代码模块称为类加载器。

 

###### 双亲委派模型

一般分为：

1、 启动类加载器：存放<JAVA_HOME>\lib目录下的类库

2、 扩展类加载器：存放<JAVA_HOME>\lib\ext目录的或者java.ext.dirs所指定的路径

3、 应用程序类加载器：加载用户路径classPath下的指定类库

 

双亲委派模型要求除了启动类加载器以为都要有父加载器。并且他们的父子关系是以组合关系来实现的。

 

工作过程：一个类加载收到类加载器请求后，会首先交给自己父类加载，父类依照上面执行，如果父类没有加载，就交给子类来加载。

 

   

#### 验证

验证是为了确保class文件字节流所包含的信息符合虚拟机要求。

 

主要分为文件格式验证、元数据验证、字节码验证、符号引用验证

1、 文件格式验证会查看字节流是否符合class文件格式规范并且能被当前虚拟机处理，之后会把字节流存储到方法区。

2、 元数据验证对字节码描述信息进行语义分析，如：

这个类是否有父类

这个类是否继承了不允许被继承的类

3、 字节码验证主要是通过数据流和控制流，确定程序语义是合法的。

4、 符号引用验证发送在虚拟机将符号引用转换为直接引用的时候。这个转换动作将在解析阶段发生，确保解析阶段顺利完成。验证内容如下：

1、符号引用中通过字符串描述的全限定名能否找到对应的类。

2、在指定类中是否存在方法的名称和字段的名称以及描述符

3、符号引用中的类、字段、方法的访问性是否可被当前类所访问。

#### 准备

正式为类变量分配内存并设置类变量初始值的阶段，这些类变量所使用的内存都将在方法区中进行分配。这里的初始值是数据类型的零值。

那这为什么不是设置好的值而是零值呢？是因为准备阶段还未调用任何方法。

也有特例那就是把类变量加final，就会在准备阶段设置为正确的值。

#### 解析

​         解析阶段是将常量池中的符号引用转换为直接引用。

 

​         一般能在类加载的时候就会把符号引用转换为直接引用的是静态方法、私有方法、实例构造器和父类方法以及被final修饰的方法。这些方法被称为非虚方法。

 

​         下面会介绍java核心特性之一多态如何实现。

​         多态的体现重载:虚拟机在重载的时候会通过参数的静态类型而不是实际类型来作为判定依据，并且静态类型是编译期可知的。

​         所以依赖静态类型来定位方法执行版本的分派动作称为静态分派。

​       多态另一个体现重写，虚拟机在运行期间根据实际类型确定方法执行版本过程称为动态分派。

#### 初始化

​         通俗说初始化阶段就是执行类构造器<clinit>

​         特点：

1、 由编译器自动采集类中的类变量复制和静态语句块（static语句块）合并产生。静态语句块只能访问到定义在之前的变量，定义在他之后的他可以为其赋值，而不能访问它。

2、 <clint>与类构造函数<init>不同。他不需要显示调用父类构造器，虚拟机保证子类构造器调用之前，父类构造器一定已经调用完了。

3、 所以父类的静态语句块优先于子类的静态语句块。

4、 接口与类不同，执行构造器时候不需要先执行父类的构造器。

5、 虚拟机保证多线程情况下<clint>的正确加锁同步。

 

 

## 动态语言和静态语言

什么叫动态语言？

它的类型检查是在运行期间而不是编译期间。

 

## Java内存模型

这里的内存模型可以理解为特定操作协议下，对特定的内存或告诉缓存进行读写访问的过程抽象。

Java虚拟机的即时编译器采用指令重排序来优化处理器。

 

 

java内存模型主要目标定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。此处变量是包括了实例、静态和构成数组对象的元素。

 

   

 

volatile关键字在一定程度上提供了同步机制。

它具有两个特性：

1、 保证此变量对所有线程可见性

在以下两个场景下不推荐使用volatile而是用synchronized等

1运算结果不依赖变量当前值

2变量不需要与其他状态的变量共同参与不变约束。

2、 禁止指令重排序

这里有一点volatile屏蔽指令重排序语义在JDK1.5才被完全修复，此前及时使用volatile也不能保证使用双锁检查来实现单例模式的原因。

 

 

 

## Jvm调优

如果出现out of memeory：Pergen space说明java虚拟机对永久代内存不够，一般是程序启动需要加载大量的第三方jar包。例如tomcat下部署太多应用，或者大量动态反射生成类的吧不断加载

 

# 算法

## 时间复杂度

O（n^2）一般处理10^4数据

O（n）一般处理10^8

O(nlogn)一般处理10^7数据

 

## 溢出

整形溢出：二分查找法中的求中间数据如  mid = (i+j)/2  i+j可能超过int而溢出

所以可以采用i+(j-i)/2

 

## 优化

一般我们需要先大概说一下时间复杂度和空间复杂度，之后根据实际来慢慢优化

 

## 方法

### 双指针

双指针主要用于遍历数组，两个指针指向不同的元素，从而协同完成任务。双指针可以从不同的方向向中间逼近也可以朝着同一个方向遍历。

题型：

[1. 有序数组的 Two Sum](http://www.cnblogs.com/#1-有序数组的-two-sum)

[2. 两数平方和](http://www.cnblogs.com/#2-两数平方和)

[3. 反转字符串中的元音字符](http://www.cnblogs.com/#3-反转字符串中的元音字符)

[4. 回文字符串](http://www.cnblogs.com/#4-回文字符串)

[5. 归并两个有序数组](http://www.cnblogs.com/#5-归并两个有序数组)

[6. 判断链表是否存在环](http://www.cnblogs.com/#6-判断链表是否存在环)

[7. 最长子序列](http://www.cnblogs.com/#7-最长子序列)

 

 

双指针的另一种变形就是滑动窗口

   

题型如：无重复字符的最长子串等

### 暴力解法

遍历所有的可能！！

 

 

### 查找问题

查找有无

元素a是否存在 set集合

 

查找对应关系

A出现多少次 map字典

 

 

 

# 面试

## 操作系统

### 进程和线程

区别：

一般一个应用程序就可以叫做进程，进程包含了很多的线程、内存空间（逻辑内存）、文件/句柄。进程与进程之间通信一般可以采用TCP/IP协议。

线程包含了调用栈（里面每个栈帧包含了返回信息和参数等信息）、程序计数器（保存了下一个字节码指令行号，属于内存空间）和TLS(线程自身内存)。线程与线程之间通信一般很方便，因为其共享了同一内存。

线程才是一个进程的实际执行者。进程更像是一个管理者或者容器。

 

### 寻址空间

与计算机的物理内存没有任何关系。其与操作系统的位数有关系。

 

 

# 编程式范式

### C语言

C语言设计目标是一种能以简易方式编译、处理底层内存，产生少了机器码以及不需要任何运行环境支持便能运行的编程语言，其运行较快并且对系统资源利用率高。但是c语言基于过程和底层的设计让他不够贴近业务和抽象。在处理泛型问题上非常蛋疼。

 

同种方法因为不同类型而导致的根据不同类型的解决方法，造成大量的无意义操作。

 

要做到泛型需要什么？

1、 标准化类型的内存分配、释放和访问

2、 标准化类型的操作。比如比较、I/O、复制操作

3、 标准化掉数据容器的操作。比如：查找算法、过滤算法等

4、 标准化掉类型上特有的操作。需要有标准化的接口来回调不同类型的具体操作

### C++

c++通过大量技术来解决了泛型问题。

 

 

 

 

# Maven

## 环节

清理:将以前编译得到的旧的class字节码删除，为下一次编译做准备

编译：将java源程序编译为class字节码

测试：自动测试，自动调用junit程序

报告：测试程序执行的结果

打包：动态web工程打war包，java工程打jar包

安装：maven特定概念，将打包和得到的文件复制到仓库中指定位置

部署：将动态web工程生成的war包，复制到tomcat容器指定目录，使其运行

 

 

## 核心

### 目录结构

\1.      根目录：工程名

\2.      Src目录：源码

\3.      Pom.xml：maven工程核心配置文件

\4.      Main目录：存放主程序

\5.      Test目录：存放测试程序

\6.      Java：源文件

\7.      Resources：存放框架或者其他工具配置文件

 

为什么要遵守上面目录结构？

Maven负责我们的自动构建，以编译为例，maven要编译就需要知道源文件放在那里。

 

 

### 常用命令

mvn clean

mvn compile

mvn test-compile

mvn test           执行测试

mvn package 打包

mvn install 安装

mvn site 生成站点

mvn deploy自动把war包放在服务器上

 

### POM

Project Object Model项目对象模型

 

### 坐标

利用下面三个向量在仓库中唯一定位一个Maven工程

Groupid：公司或组织域名倒叙+项目名

Artifactid：模块名

Version：版本

 

### 仓库

本地仓库：本地电脑上部署的仓库

远程仓库：1、私服：私人或者公司自己搭建的仓库

3、 中央仓库：阿里云和maven官方等仓库

 

 

### 依赖

Maven解析依赖信息时回到本地仓库查找所需要的依赖

​                  对于我们自己开发的jar，我们就用mvn install安装到本地仓库

 

#### 范围

   

 

1、 Compile

对主程序是否有效：是

对测试程序是否有效：是

是否参与打包：是

   

2、 Test

对主程序是否有效：否

对测试程序是否有效：有

是否参与打包：否

3、 Provided

对主程序是否有效：是

对测试程序是否有效：是

是否参与打包：否

是否参与部署：否

 

   

 

 

#### 传递性

1、 可以传递的依赖不必每次在模块中声明，在最下面的模块中声明即可

2、 非compile范围内的依赖不能传递，所以各个模块有需要需要重复依赖

#### 排除

   排出点引入依赖里面不需要的依赖

 

#### 原则

1、 路径最优者有限

2、 先声明者优先

#### 统一管理依赖版本

要对spring各个jar同一升级版本

1、 properties标签内使用自动以标签统一声明版本号

2、 在需要统一版本的地方，使用${自动以标签}引用声明的版本号

   

 

 

 

### 继承

需要：统一管理各个模块工程对junit的依赖，由于是test范围不能传递，那么各个模块版本很可能不一致

 

解决：将junit依赖版本统一提取到父工程中，在子依赖声明中不指定版本

 

操作步骤：

1、 创建一个maven工程，注意打包方式是pom

2、 在子工程中声明对父工程的引用<parent>父工程坐标体系</parent>

3、 将子工程坐标中与父工程坐标重复的内容删除

4、 在父工程中统一声明junit依赖

   

5、 在子工程中删除junit依赖版本号

 

配置继承后，需要先安装父工程否则子工程无法安装

 

 

### 聚合

就是一键安装各个模块工程

 

配置方式：在一个总的聚合工程中去配置各个参与聚合的模块

   

 

使用方式：mvn install 父工程

 

### 生命周期

各个构建环节的顺序执行：不能打乱顺序。

 

Maven分为clean，defalut和site三个生命周期

 

#### Default

清理-编译-测试-打包-安装-部署

都是从最开始往后执行，从上到下

 

 

### 自动部署到服务器

<build>

<plugins>

​         <plugin>

​                  //添加cargo 是启动servlet容器

​                  <configuration>

​                  <container>tomcat

​         </plugin>

</plugins>

</build>

 

# 计算机基础

分层

   

 

## 数据链路层

### 三个问题

封装成帧、透明传输、差错控制

#### 封装成帧

将ip数据报添加上首部尾部来做一个界限

 

#### 透明传输

为了解决数据中出现开始和结束限定符，采用字符填充来解决，也就是透明传输

在数据中出现开始和结束界定符时候，在前面添加ESC来与开始和结束区别。

 

### 差错控制

CRC循环冗余检验

除数随意选择，但要比被除数后面划线的零多一位。

得到的余数代替被除数最后的零。

 

之后把被除数和除数传给另一台计算机，如果得到商为0，则为正确的数据

   

 

CRC是一种常用检查方法

CRC是fcs的一种方法

CRC是无差错接受，而不是无传输差错

 

## 以太网

### CSMA/CD协议

CSMA/CD 载波监听多点接入/碰撞检测

多点接入：表示许多计算机以多点接入连接在一条总线上

载波监听：发送数据前探测总线上是否有数据在传输，有就先等待。

 

## TCP/IP

Tcp是在ip层不可靠的协议上加了可靠地运输层（tcp的重传机制）

Udp是不可靠的传输协议，但不需要建立连接。

 

## Arp

Ip地址转换为mac地址

原理：通过广播获得mac地址

 

缺点：容器被截获并欺骗

 

 

 

 

 

 

 

 

 

 

 

 

 

​                                                                                                                                                                                                                                                                                                                                                                        