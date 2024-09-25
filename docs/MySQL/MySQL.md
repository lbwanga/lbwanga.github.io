[MySQL :: MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)



## select 语句的执行过程

![](MySQL\基础架构.png)



### 连接器

作用：

1. 与客户端进行TCP三次握手建立连接
2. 校验用户名、密码
3. 校验权限



`MySQL`也会“安排”一条线程维护当前客户端的连接，这条线程也会时刻标识着当前连接在干什么工作，可以通过`show processlist;`命令查询所有正在运行的线程。



MySQL的最大线程数可以通过参数`max-connections`来控制，如果到来的客户端连接超出该值时，新到来的连接都会被拒绝，关于最大连接数的一些命令主要有两条：

- `show variables like '%max_connections%';`：查询目前`DB`的最大连接数。默认151
- `set GLOBAL max_connections = 200;`：修改数据库的最大连接数为指定值。



MySQL 定义了空闲连接的最大空闲时长，由 `wait_timeout` 参数控制的，默认值是 8 小时（28880秒），如果空闲连接超过了这个时间，连接器就会自动将它断开。

一个处于空闲状态的连接被服务端主动断开后，这个客户端并不会马上知道，等到客户端在发起下一个请求的时候，才会收到这样的报错“ERROR 2013 (HY000): Lost connection to MySQL server during query”。



MySQL 的连接也跟 HTTP 一样，有短连接和长连接的概念。长连接的好处就是可以减少建立连接和断开连接的过程，但是，使用长连接后可能会占用内存增多。有两种解决方式：

第一种，**定期断开长连接**。既然断开连接后就会释放连接占用的内存资源，那么我们可以定期断开长连接。

第二种，**客户端主动重置连接**。MySQL 5.7 版本实现了 `mysql_reset_connection()` 函数的接口，注意这是接口函数不是命令，那么当客户端执行了一个很大的操作后，在代码里调用 mysql_reset_connection 函数来重置连接，达到释放内存的效果。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。



### 查询缓存

连接器的工作完成后，客户端就可以向 MySQL 服务发送 SQL 语句了，MySQL 服务收到 SQL 语句后，就会解析出 SQL 语句的第一个字段，看看是什么类型的语句。

如果 SQL 是查询语句（select 语句），MySQL 就会先去查询缓存（ Query Cache ）里查找缓存数据，看看之前有没有执行过这一条命令，这个查询缓存是以 key-value 形式保存在内存中的，key 为 SQL 查询语句，value 为 SQL 语句查询的结果。

MySQL 8.0 版本已经将查询缓存删掉。



### 解析SQL

第一件事情，**词法分析**。MySQL 会根据你输入的字符串识别出关键字出来，例如，SQL语句 select username from userinfo，在分析之后，会得到4个Token，其中有2个Keyword，分别为select和from。

第二件事情，**语法分析**。根据词法分析的结果，语法解析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法，如果没问题就会构建出 SQL 语法树，这样方便后面模块获取 SQL 类型、表名、字段名、 where 条件等等。



`SQL`语句分为五大类：

- `DML`：数据库操作语句，比如`update、delete、insert`等都属于这个分类。
- `DDL`：数据库定义语句，比如`create、alter、drop`等都属于这个分类。
- `DQL`：数据库查询语句，比如最常见的`select`就属于这个分类。
- `DCL`：数据库控制语句，比如`grant、revoke`控制权限的语句都属于这个分类。
- `TCL`：事务控制语句，例如`commit、rollback、setpoint`等语句属于这个分类。



存储过程：是指提前编写好的一段较为常用或复杂`SQL`语句，然后指定一个名称存储起来，然后先经过编译、优化，完成后，这个“过程”会被嵌入到`MySQL`中。



触发器则是一种特殊的存储过程，但[触发器]与[存储过程]的不同点在于：**存储过程需要手动调用后才可执行，而触发器可由某个事件主动触发执行**。在`MySQL`中支持`INSERT、UPDATE、DELETE`三种事件触发，同时也可以通过`AFTER、BEFORE`语句声明触发的时机，是在操作执行之前还是执行之后。



### 执行SQL

#### 预处理阶段

检查 SQL 查询语句中的表或者字段是否存在；

将 `select *` 中的 `*` 符号，扩展为表上的所有列；

#### 优化阶段

**优化器主要负责将 SQL 查询语句的执行方案确定下来**，比如在表里面有多个索引的时候，优化器会基于查询成本的考虑，来决定选择使用哪个索引。

`MySQL`优化器的一些优化准则如下：

- 多条件查询时，重排条件先后顺序，将效率更好的字段条件放在前面。

- 当表中存在多个索引时，选择效率最高的索引作为本次查询的目标索引。

- 使用分页`Limit`关键字时，查询到对应的数据条数后终止扫表。

- 多表`join`联查时，对查询表的顺序重新定义，同样以效率为准。

- 对于`SQL`中使用函数时，如`count()、max()、min()...`，根据情况选择最优方案。

- - `max()`函数：走`B+`树最右侧的节点查询（大的在右，小的在左）。
  - `min()`函数：走`B+`树最左侧的节点查询。
  - `count()`函数：如果是`MyISAM`引擎，直接获取引擎统计的总行数。

- 对于`group by`分组排序，会先查询所有数据后再统一排序，而不是一开始就排序。



#### 执行阶段

在执行的过程中，执行器就会和存储引擎交互，交互是以记录为单位的。



## update 语句的执行过程

查询语句的那一套流程，更新语句也是同样会走一遍：

1. 客户端先通过连接器建立连接，连接器自会判断用户身份、权限校验；
2. 因为这是一条 update 语句，所以不需要经过查询缓存，但是表上有更新语句，是会把整个表的查询缓存清空的，所以说查询缓存很鸡肋，在 MySQL 8.0 就被移除这个功能了；
3. 解析器会通过词法分析识别出关键字 update，表名等等，构建出语法树，接着还会做语法分析，判断输入的语句是否符合 MySQL 语法；
4. 预处理器会判断表和字段是否存在；
5. 优化器确定执行计划；
6. 执行器负责具体执行。

与查询流程不一样的是，更新流程还涉及两个重要日志模块：redo log（重做日志）和 bin log（归档日志）。

update执行流程：` update T set c=c+1 where ID=2;` 

7. 执行器先找引擎取ID=2这一行。如果ID=2这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。

8. 执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

9. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 undo-log 和 redo log（prepare状态）里面。然后告知执行器执行完成了，随时可以提交事务。

10. 执行器生成这个操作的binlog，并把binlog写入磁盘。

11. 执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交（commit）状态，更新完成。

![](MySQL\update.png)

图中浅色框表示是在InnoDB内部执行的，深色框表示是在执行器中执行的。

将 redo log 的写入拆成了两个步骤： prepare 和 commit，这就是"两阶段提交"。为了使两个日志之间保持一致：

1. 当在写bin log之前崩溃时：此时 binlog 还没写，redo log 也还没提交，事务会回滚。 日志保持一致 

2. 当在写bin log之后崩溃时： 重启恢复后虽没有commit，但满足prepare和binlog完整，自动commit。日志保持一致 

溃恢复时的判断规则：

1. 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交； 
2. 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
   1. 如果是，则提交事务； 
   2. 否则，回滚事务。



## 存储结构

先来看看 MySQL 数据库的文件存放在哪个目录？

```sh
mysql> SHOW VARIABLES LIKE 'datadir';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+
1 row in set (0.00 sec)
```

我们每创建一个 database（数据库） 都会在 /var/lib/mysql/ 目录里面创建一个以 database 为名的目录，然后保存表结构和表数据的文件都会存放在这个目录里。

- db.opt，用来存储当前数据库的默认字符集和字符校验规则。
- 表名.frm ，数据库表的**表结构**会保存在这个文件。在 MySQL 中建立一张表都会生成一个.frm 文件，该文件是用来保存每个表的元数据信息的，主要包含表结构定义。
- 表名.ibd，数据库表的**表数据**会保存在这个文件。 MySQL 中每一张表的数据都存放在一个独立的 .ibd 文件。



### 表空间

**表空间由段（segment）、区（extent）、页（page）、行（row）组成**，InnoDB存储引擎的逻辑存储结构大致如下图：

![](.\MySql\表空间结构.drawio.webp)

> 记录是按照行来存储的
>
> InnoDB 的数据是按「页」为单位来读写的，默认每个页的大小为 16KB。一次最少从磁盘中读取 16K 的内容到内存中，一次最少把内存中的 16K 内容刷新到磁盘中。



**区（extent）**

我们知道 InnoDB 存储引擎是用 B+ 树来组织数据的。

B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机I/O，随机 I/O 是非常慢的。

解决这个问题也很简单，就是让链表中相邻的页的物理位置也相邻，这样就可以使用顺序 I/O 了，那么在范围查询（扫描叶子节点）的时候性能就会很高。

那具体怎么解决呢？

在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了.



表空间是由各个段（segment）组成的，段是由多个区（extent）组成的。段一般分为数据段、索引段和回滚段等。



### InnoDB 行格式

InnoDB 提供了 4 种行格式，分别是 Redundant、Compact、Dynamic和 Compressed 行格式。

Compact 行格式：

![](.\MySql\COMPACT.drawio.webp)

#### 记录的额外信息

##### **变长字段长度列表**

varchar(n) 和 char(n) 的区别是char 是定长的，varchar 是变长的，变长字段实际存储的数据的长度（大小）是不固定的。

所以，在存储数据的时候，也要把数据占用的大小存起来，存到「变长字段长度列表」里面，读取数据的时候才能根据这个「变长字段长度列表」去读取对应长度的数据。其他 TEXT、BLOB 等变长字段也是这么实现的。

![](.\MySql\t_test.webp)

第一条记录：

- name 列的值为 a，真实数据占用的字节数是 1 字节，十六进制 0x01；
- phone 列的值为 123，真实数据占用的字节数是 3 字节，十六进制 0x03；
- age 列和 id 列不是变长字段，所以这里不用管。

这些变长字段的真实数据占用的字节数会按照列的顺序**逆序存放**，所以「变长字段长度列表」里的内容是「 03 01」

![](.\MySql\变长字段长度列表1.webp)



> **为什么「变长字段长度列表」的信息要按照逆序存放？**

这个设计是有想法的，主要是因为「记录头信息」中指向下一个记录的指针，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。

「变长字段长度列表」中的信息之所以要逆序存放，是因为这样可以**使得位置靠前的记录的真实数据和数据对应的字段长度信息可以同时在一个 CPU Cache Line 中，这样就可以提高 CPU Cache 的命中率**。



> **每个数据库表的行格式都有「变长字段字节数列表」吗？**

**当数据表没有变长字段的时候，比如全部都是 int 类型的字段，这时候表里的行格式就不会有「变长字段长度列表」了**，因为没必要，不如去掉以节省空间。



##### NULL值列表

表中的某些列可能会存储 NULL 值，如果把这些 NULL 值都放到记录的真实数据中会比较浪费空间，所以 Compact 行格式把这些值为 NULL 的列存储到 NULL值列表中。

如果存在允许 NULL 值的列，则每个列对应一个二进制位（bit），二进制位按照列的顺序**逆序排列**。

- 二进制位的值为`1`时，代表该列的值为NULL。
- 二进制位的值为`0`时，代表该列的值不为NULL。

另外，NULL 值列表必须用**整数个字节**的位表示（1字节8位），如果使用的二进制位个数不足整数个字节，则在字节的高位补 `0`。

第三条记录 phone 列 和 age 列是 NULL 值，所以，对于第三条数据，NULL 值列表用十六进制表示是 0x06。

![](.\MySql\null值列表4.webp)

> **每个数据库表的行格式都有「NULL 值列表」吗？**

**当数据表的字段都定义成 NOT NULL 的时候，这时候表里的行格式就不会有 NULL 值列表了**。

所以在设计数据库表的时候，通常都是建议将字段设置为 NOT NULL，这样可以至少节省 1 字节的空间



##### 记录头信息

记录头信息中包含的内容很多：

- delete_mask ：标识此条数据是否被删除。从这里可以知道，我们执行 detele 删除记录的时候，并不会真正的删除记录，只是将这个记录的 delete_mask 标记为 1。
- next_record：下一条记录的位置。从这里可以知道，记录与记录之间是通过链表组织的。在前面我也提到了，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。
- record_type：表示当前记录的类型，0表示普通记录，1表示B+树非叶子节点记录，2表示最小记录，3表示最大记录



#### 记录的真实数据

记录真实数据部分除了我们定义的字段，还有三个隐藏字段，分别为：row_id、trx_id、roll_pointer。

* row_id：如果我们建表的时候指定了主键或者唯一索引，那么就没有 row_id 隐藏字段了。如果既没有指定主键，又没有唯一索引，那么 InnoDB 就会为记录添加 row_id 隐藏字段。row_id不是必需的，占用 6 个字节。

- trx_id：事务id，表示这个数据是由哪个事务生成的。 trx_id是必需的，占用 6 个字节。

- roll_pointer：这条记录上一个版本的指针。roll_pointer 是必需的，占用 7 个字节。



### varchar(n) 中 n 最大取值为多少？

**MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节**。

要算 varchar(n) 最大能允许存储的字节数，还要看数据库表的字符集，因为字符集代表着，1个字符要占用多少字节，比如 ascii 字符集， 1 个字符占用 1 字节。

存储字段类型为 varchar(n) 的数据时，其实分成了三个部分来存储：

- 真实数据
- 真实数据占用的字节数
- NULL 标识，如果不允许为NULL，这部分不需要

所以，我们在算 varchar(n) 中 n 最大值时，需要减去 「变长字段长度列表」和 「NULL 值列表」所占用的字节数的。



```sql
CREATE TABLE test ( 
`name` VARCHAR(65532)  NULL
) ENGINE = InnoDB DEFAULT CHARACTER SET = ascii ROW_FORMAT = COMPACT;
```

上述例子，在数据库表只有一个 varchar(n) 字段且字符集是 ascii 的情况下，varchar(n) 中 n 最大值 = 65535 - 2 - 1 = 65532。



### 行溢出后，MySQL 是怎么处理的？

MySQL 中磁盘和内存交互的基本单位是页，一个页的大小一般是 `16KB`，也就是 `16384字节`，而一个 varchar(n) 类型的列最多可以存储 `65532字节`，一些大对象如 TEXT、BLOB 可能存储更多的数据，这时一个页可能就存不了一条记录。这个时候就会**发生行溢出，多的数据就会存到另外的「溢出页」中**。

当发生行溢出时，在记录的真实数据处只会保存该列的一部分数据，而把剩余的数据放在「溢出页」中，然后真实数据处用 20 字节存储指向溢出页的地址，从而可以找到剩余数据所在的页.



## 索引

### 分类

按「数据结构」分类：B+tree索引、Hash索引、Full-text索引。

按「物理存储」分类：聚簇索引（主键索引）、二级索引（辅助索引）。

按「字段特性」分类：主键索引、唯一索引、普通索引、前缀索引。

按「字段个数」分类：单列索引、联合索引。

#### B+Tree索引

***1、B+Tree vs 二叉树***

对于有 N 个叶子节点的 B+Tree，其搜索复杂度为`O(logdN)`，其中 d 表示节点允许的最大子节点个数为 d 个。

在实际的应用当中， d 值是大于100的，这样就保证了，即使数据达到千万级别时，B+Tree 的高度依然维持在 3~4 层左右，也就是说一次数据查询操作只需要做 3~4 次的磁盘 I/O 操作就能查询到目标数据。

而二叉树的每个父节点的儿子节点个数只能是 2 个，意味着其搜索复杂度为 `O(logN)`，这已经比 B+Tree 高出不少，因此二叉树检索到目标数据所经历的磁盘 I/O 次数要更多。如果索引的字段值是按顺序增长的，二叉树会转变为链表结构，检索的过程和全表扫描无异。

**2、B+Tree vs 红黑树**

红黑树虽然对比二叉树来说，树高有所降低，但数据量一大时，依旧会有很大的高度。每个节点中只存储一个数据，节点之间还是不连续的，依旧无法利用局部性原理。

***3、B+Tree vs B Tree***

B+Tree 只在叶子节点存储数据，而 B 树 的非叶子节点也要存储数据，所以 B+Tree 的单个节点的数据量更小，在相同的**磁盘 I/O 次数**下，就能查询更多的节点。

另外，B+Tree 叶子节点采用的是双链表连接，适合 MySQL 中常见的**范围查询**，而 B 树无法做到这一点。

B+Tree**插入和删除效率更高**，不会涉及复杂的树的变形

***4、B+Tree vs Hash***

Hash 在做等值查询的时候效率贼快，搜索复杂度为 O(1)。

但是 Hash 表不适合做范围查询，它更适合做等值的查询，这也是 B+Tree 索引要比 Hash 表索引有着更广泛的适用场景的原因。

#### 聚集索引和二级索引

![](MySQL\聚集索引和二级索引.webp)

在查询时使用了二级索引，如果查询的数据能在二级索引里查询的到，那么就不需要回表，这个过程就是覆盖索引。

如果查询的数据不在二级索引里，就会先检索二级索引，找到对应的叶子节点，获取到主键值后，然后再检索主键索引查询到数据，这个过程就是回表。

#### 主键索引和唯一索引

一张表最多只有一个主键索引，索引列的值不允许有空值。

唯一索引建立在 UNIQUE 字段上的索引，一张表可以有多个唯一索引，索引列的值必须唯一，但是允许有空值。

#### 前缀索引

前缀索引的特点是短小精悍，我们可以利用一个字段的前`N`个字符创建索引，相较于使用一个完整字段创建索引，前缀索引能够更加节省存储空间。

但是无法通过前缀索引来完成`ORDER BY、GROUP BY`等分组排序工作，同时也无法完成覆盖扫描等操作。



#### 联合索引

使用联合索引时，存在**最左匹配原则**，也就是按照最左优先的方式进行索引的匹配。在使用联合索引进行查询的时候，如果不遵循「最左匹配原则」，联合索引会失效。

联合索引有一些特殊情况，并不是查询过程使用了联合索引查询，就代表联合索引中的所有字段都用到了联合索引进行索引查询。联合索引的最左匹配原则会一直向右匹配直到遇到「范围查询」就会停止匹配。**也就是范围查询的字段可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引**。



> **`select * from t_table where a > 1 and b = 2`，联合索引（a, b）哪一个字段用到了联合索引的 B+Tree？**

由于联合索引（二级索引）是先按照 a 字段的值排序的，所以符合 a > 1 条件的二级索引记录肯定是相邻，于是在进行索引扫描的时候，可以定位到符合 a > 1 条件的第一条记录，然后沿着记录所在的链表向后扫描，直到某条记录不符合 a > 1 条件位置。所以 a 字段可以在联合索引的 B+Tree 中进行索引查询。

但是在符合 a > 1 条件的二级索引记录的范围里，b 字段的值是无序的。所以 b 字段无法利用联合索引进行索引查询。

这条查询语句只有 a 字段用到了联合索引进行索引查询，而 b 字段并没有使用到联合索引。



> **`select * from t_table where a >= 1 and b = 2`，联合索引（a, b）哪一个字段用到了联合索引的 B+Tree？**

由于联合索引（二级索引）是先按照 a 字段的值排序的，所以符合 >= 1 条件的二级索引记录肯定是相邻，于是在进行索引扫描的时候，可以定位到符合 >= 1 条件的第一条记录，然后沿着记录所在的链表向后扫描，直到某条记录不符合 a>= 1 条件位置。所以 a 字段可以在联合索引的 B+Tree 中进行索引查询。

虽然在符合 a>= 1 条件的二级索引记录的范围里，b 字段的值是「无序」的，但是对于符合 a = 1 的二级索引记录的范围里，b 字段的值是「有序」的.

于是，在确定需要扫描的二级索引的范围时，当二级索引记录的 a 字段值为 1 时，可以通过 b = 2 条件减少需要扫描的二级索引记录范围。也就是说，从符合 a = 1 and b = 2 条件的第一条记录开始扫描，而不需要从第一个 a 字段值为 1 的记录开始扫描。

所以，这条查询语句 a 和 b 字段都用到了联合索引进行索引查询。



> **`SELECT * FROM t_table WHERE a BETWEEN 2 AND 8 AND b = 2`，联合索引（a, b）哪一个字段用到了联合索引的 B+Tree？**

在 MySQL 中，BETWEEN 包含了 value1 和 value2 边界值，类似于 >= and =<，所以这条查询语句 a 和 b 字段都用到了联合索引进行索引查询。



> **`SELECT * FROM t_user WHERE name like 'j%' and age = 22`，联合索引（name, age）哪一个字段用到了联合索引的 B+Tree？**

由于联合索引（二级索引）是先按照 name 字段的值排序的，所以前缀为 ‘j’ 的 name 字段的二级索引记录都是相邻的， 于是在进行索引扫描的时候，可以定位到符合前缀为 ‘j’ 的 name 字段的第一条记录，然后沿着记录所在的链表向后扫描，直到某条记录的 name 前缀不为 ‘j’ 为止。

虽然在符合前缀为 ‘j’ 的 name 字段的二级索引记录的范围里，age 字段的值是「无序」的，但是对于符合 name = j 的二级索引记录的范围里，age字段的值是「有序」的

所以，这条查询语句 a 和 b 字段都用到了联合索引进行索引查询。



**联合索引的最左匹配原则，在遇到范围查询（如 >、<）的时候，就会停止匹配，也就是范围查询的字段可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引。注意，对于 >=、<=、BETWEEN、like 前缀匹配的范围查询，并不会停止匹配**。



**联合索引进行排序**：

这里出一个题目，针对针对下面这条 SQL，你怎么通过索引来提高查询效率呢？

```sql
select * from order where status = 1 order by create_time asc
```

有的同学会认为，单独给 status 建立一个索引就可以了。

但是更好的方式给 status 和 create_time 列建立一个联合索引，因为这样可以避免 MySQL 数据库发生文件排序。

因为在查询时，如果只用到 status 的索引，但是这条语句还要对 create_time 排序，这时就要用文件排序 filesort，也就是在 SQL 执行计划中，Extra 列会出现 Using filesort。

所以，要利用索引的有序性，在 status 和 create_time 列建立联合索引，这样根据 status 筛选后的数据就是按照 create_time 排好序的，避免在文件排序，提高了查询效率。



### 索引设计原则

什么时候适合索引？

1. 针对数据量较大，且查询比较繁琐的表建立索引；

2. 针对于常作为查询条件（where），排序（order by），分组(group by)操作的字段，建立索引；

3. 尽量选择区分度高的列作为索引，尽量建立唯一索引，区分度越高使用索引的效率越高；

4. 如果是字符串类型的字段，字段的长度过长，可以针对字段的特点，建立前缀索引；

5. 建立联合索引，应当遵循最左前缀原则，将多个字段之间按优先级顺序组合；

6. 尽量使用联合索引，减少单列索引，查询时，联合索引很多时候可以覆盖索引，节省存储空间，避免回表，提高查询效率；

7. 要控制索引的数量，索引并不是多多益善，索引越多，维护索引结构的代价也就越大，会影响增删改的效率；

8. 如果索引列不能存储null值，在创建表时使用not null约束它。当优化器知道每列是否包含null值时，它可以更好地确定哪个索引最有效地用于查询。

9. 表的主外键或连表字段，必须建立索引，因为能很大程度提升连表查询的性能。



什么时候不适合索引？

1. 大量重复值的字段
2. 当表的数据较少，不应当建立索引，因为数据量不大时，维护索引反而开销更大。
3. 经常增删改的字段，因为索引字段频繁修改，由于要维护 B+Tree的有序性，那么就需要频繁的重建索引，这个过程是会影响数据库性能。
4. 索引不能参与计算，因此经常带函数查询的字段，并不适合建立索引。
5. 一张表中的索引数量并不是越多越好，一般控制在`3`，最多不能超过`5`。
6. 索引的字段值无序时，不推荐建立索引，因为会造成页分裂，尤其是主键索引。



### 索引失效

1. 左或左右模糊查询 `like %x 或者 like %x%`。 因为索引 B+ 树是按照「索引值」有序排列存储的，只能根据前缀进行比较。
2. 查询中对索引做了计算、函数、类型转换操作。因为索引保存的是索引字段的原始值，而不是经过函数计算后的值，自然就没办法走索引了。
3. 联合索引要遵循最左匹配原则
4. 联合索引中，出现范围查询（>,<），范围查询右侧的列索引失效。
5. 在 WHERE 子句中，如果 OR 前后有条件列不是索引列，那么索引会失效。因为 OR 的含义就是两个只要满足一个即可，只要有条件列不是索引列，就会进行全表扫描。
6. 隐式类型转换



**索引隐式类型转换**：

如果索引字段是字符串类型，但是在条件查询中，输入的参数是整型的话，你会在执行计划的结果发现这条语句会走全表扫描；

但是如果索引字段是整型类型，查询条件中的输入参数即使是字符串，也不会导致索引失效，还是可以走索引扫描。

MySQL 在遇到字符串和数字比较的时候，会自动把字符串转为数字，然后再进行比较。



### 索引下推

对于联合索引（a, b），在执行 `select * from table where a > 1 and b = 2` 语句的时候，只有 a 字段能用到索引，那在联合索引的 B+Tree 找到第一个满足条件的主键值后，还需要判断其他条件是否满足（看 b 是否等于 2），那是在联合索引里判断？还是回主键索引去判断呢？

- 在 MySQL 5.6 之前，只能一个个回表，返回给server层再对比 b 字段值。
- 而 MySQL 5.6 引入的**索引下推优化**（index condition pushdown)， 可以在联合索引遍历过程中，对联合索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。



## 事务

**事务的特性**：

* 原子性：一个事务中的所有操作，要么全部完成，要么全部失败。

* 一致性：是指事务操作前和操作后，数据满足完整性约束，数据库保持一致性状态。

* 隔离性：多个事务同时使用相同的数据时，不会相互干扰，每个事务都有一个完整的数据空间，对其他并发事务是隔离的。

* 持久性：一个事务一旦被提交，它会保持永久性，所更改的数据都会被写入到磁盘做持久化处理。



**InnoDB 引擎通过什么技术来保证事务的这四个特性的呢？**

- 持久性是通过 redo log （重做日志）来保证的，宕机后能数据恢复；
- 原子性是通过 undo log（回滚日志） 来保证的，事务能够进行回滚；
- 隔离性是通过 MVCC（多版本并发控制） 或锁机制来保证的；
- 一致性则是通过持久性+原子性+隔离性来保证；



**并行事务会引发的问题**：

* 脏读：读到其他事务未提交的数据；

* 不可重复读：一个事务内，前后读取的数据不一致；

* 幻读：一个事务中，前后读取的记录数量不一致。事务中同一个查询在不同的时间产生不同的结果集。

严重性：脏读 > 不可重读读 > 幻读



**隔离级别**：

- 读未提交（read uncommitted），指一个事务还没提交时，它做的变更就能被其他事务看到；
- 读提交（read committed），指一个事务提交之后，它做的变更才能被其他事务看到；
- 可重复读（repeatable read），指一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，MySQL InnoDB 引擎的默认隔离级别；
- 串行化（serializable）；会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行；

![](./MySql/隔离级别.png)



**这四种隔离级别具体是如何实现的呢？**

- 对于「读未提交」隔离级别的事务来说，写操作加排他锁，读操作不加锁；
- 对于「串行化」隔离级别的事务来说，所有写操作加临键锁，所有读操作加共享锁；
- 对于「读提交」和「可重复读」隔离级别的事务来说，它们是通过 **Read View 来实现的，它们的区别在于创建 Read View 的时机不同，「读提交」隔离级别是在「每个语句执行前」都会重新生成一个 Read View，而「可重复读」隔离级别是「启动事务时」生成一个 Read View，然后整个事务期间都在用这个 Read View**。



### MVCC

在`MySQL`众多的开源存储引擎中，几乎只有`InnoDB`实现了`MVCC`机制，仅在`RC`读已提交级别、`RR`可重复读级别才会使用`MVCC`机制。

多版本主要依赖`Undo-log`日志来实现，而并发控制则通过表的隐藏字段+`ReadView`快照来实现。当一个事务尝试改动某条数据时，会将原本表中的旧数据放入`Undo-log`日志中；当一个事务尝试查询某条数据时，`MVCC`会生成一个`ReadView`快照。

Read View 中的字段：

* `m_ids`：表示在生成当前`ReadView`时，系统内活跃的事务`ID`列表。启动但还未提交的事务ID
* `min_trx_id`：活跃的事务列表中，最小的事务`ID`。
* `max_trx_id`：表示在生成当前`ReadView`时，系统中要给下一个事务分配的`ID`值。
* `creator_trx_id`：代表创建当前这个`ReadView`的事务`ID`。

聚簇索引记录中两个跟事务有关的隐藏列：

* `TRX_ID`：最近一次改动当前这条数据的事务`ID`
* `ROLL_PTR`：回滚指针。当一个事务对一条数据做了改动后，都会将旧版本的数据放到`Undo-log`日志中，而`ROLL_PTR`就是一个地址指针，指向`Undo-log`日志中旧版本的数据，当需要回滚事务时，就可以通过这个隐藏列，来找到改动之前的旧版本数据。

![](./MySql/mvcc.webp)

一个事务去访问记录的时候，除了自己的更新记录总是可见之外，还有这几种情况：

- 如果记录的 trx_id 值小于 Read View 中的 `min_trx_id` 值，表示这个版本的记录是在创建 Read View **前**已经提交的事务生成的，所以该版本的记录对当前事务**可见**。
- 如果记录的 trx_id 值大于等于 Read View 中的 `max_trx_id` 值，表示这个版本的记录是在创建 Read View **后**才启动的事务生成的，所以该版本的记录对当前事务**不可见**。
- 如果在二者之间，需要判断 trx_id 是否在 m_ids 列表中：
  - 如果记录的 trx_id **在** `m_ids` 列表中，表示生成该版本记录的活跃事务依然活跃着（还没提交事务），所以该版本的记录对当前事务**不可见**。
  - 如果记录的 trx_id **不在** `m_ids`列表中，表示生成该版本记录的活跃事务已经被提交，所以该版本的记录对当前事务**可见**。



>  **可重复读如何工作的？**

**可重复读隔离级别是启动事务时生成一个 Read View，然后整个事务期间都在用这个 Read View**。

如果记录中的 trx_id 小于 Read View中的最小事务id，或者在最小事务id和下一个事务id之间并且不在活跃事务id中，则直接读取记录值；

如果记录中的 trx_id 在最小事务id和下一个事务id之间并且在活跃事务id中，则沿着 undo log 旧版本指针往下找旧版本的记录，直到找到 trx_id 「小于」当前事务 的 Read View 中的 min_trx_id 值的第一条记录。



> **读已提交如何工作的？**

**读提交隔离级别是在每次读取数据时，都会生成一个新的 Read View**。



> **MySQL 可重复读和幻读**

MySQL InnoDB 引擎的默认隔离级别虽然是「可重复读」，但是它很大程度上避免幻读现象（并不是完全解决了），解决的方案有两种：

- 针对**快照读**（普通 select 语句），是**通过 MVCC 方式解决了幻读**，因为可重复读隔离级别下，事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，即使中途有其他事务插入了一条数据，是查询不出来这条数据的，所以就很好了避免幻读问题。
- 针对**当前读**（select ... for update 等语句，会读取最新的数据），是**通过 next-key lock（记录锁+间隙锁）方式解决了幻读**，因为当执行 select ... for update 语句的时候，会加上 next-key lock，如果有其他事务在 next-key lock 锁范围内执行增、删、改时，就会阻塞，所以就很好了避免幻读问题。



>  **MySQL Innodb 中的 MVCC 并不能完全避免幻读现象**

第一个发生幻读现象的场景：

在可重复读隔离级别下，事务 A 第一次执行普通的 select 语句时生成了一个 ReadView，之后事务 B 向表中新插入了一条 id = 5 的记录并提交。接着，事务 A 对 id = 5 这条记录进行了更新操作，在这个时刻，这条新记录的 trx_id 隐藏列的值就变成了事务 A 的事务 id，之后事务 A 再使用普通 select 语句去查询这条记录时就可以看到这条记录了，于是就发生了幻读。

![](.\MySql\幻读发生.drawio.webp)

第二个发生幻读现象的场景：

T1 时刻：事务 A 先执行「快照读语句」：select * from t_test where id > 100 得到了 3 条记录。

T2 时刻：事务 B 往插入一个 id= 200 的记录并提交；

T3 时刻：事务 A 再执行「当前读语句」 select * from t_test where id > 100 for update 就会得到 4 条记录，此时也发生了幻读现象。

**要避免这类特殊场景下发生幻读的现象的话，就是尽量在开启事务之后，马上执行 select ... for update 这类当前读的语句**，因为它会对记录加 next-key lock，从而避免其他事务插入一条新记录。



## 锁

### 全局锁

使用全局锁 `flush tables with read lock` 后数据库处于只读状态，`unlock tables` 释放全局锁，会话断开全局锁自动释放。

应用场景：全库逻辑备份

如果数据库的引擎支持的事务支持**可重复读的隔离级别**，那么在备份数据库之前先开启事务，会先创建 Read View，然后整个事务执行期间都在用这个 Read View，而且由于 MVCC 的支持，备份期间业务依然可以对数据进行更新操作。

备份数据库的工具是 mysqldump，在使用 mysqldump 时加上 `–single-transaction` 参数的时候，就会在备份数据库之前先开启事务。这种方法只适用于支持「可重复读隔离级别的事务」的存储引擎。



### 表级锁

表级锁包括表锁、元数据锁、意向锁、自增锁

**表锁**

```sh
//表级别的共享锁，也就是读锁；
lock tables t_student read;

//表级别的独占锁，也就是写锁；
lock tables t_stuent write;

// 释放会话所有表锁，会话退出后，也会释放所有表锁
unlock tables;
```

表锁除了会限制别的线程的读写外，也会限制本线程接下来的读写操作。表锁的颗粒度太大，尽量避免使用。



**元数据锁（Meta Data Lock）**

我们不需要显示的使用 MDL，因为当我们对数据库表进行操作时，会自动给这个表加上 MDL。

MDL 是为了保证当用户对表执行 CRUD 操作时，防止其他线程对这个表结构做了变更。

对一张表进行 CRUD 操作时，加的是 MDL 读锁；对一张表做结构变更操作的时候，加的是 MDL 写锁；

MDL 是在事务提交后才会释放，事务执行期间，MDL 是一直持有的。

申请 MDL 锁的操作会形成一个队列，队列中写锁获取优先级高于读锁，一旦出现 MDL 写锁等待，会阻塞后续该表的所有 CRUD 操作。



**意向锁（Intention Lock）**

如果没有「意向锁」，那么加「独占表锁」时，就需要遍历表里所有记录，查看是否有记录存在独占锁，这样效率会很慢。

那么有了「意向锁」，由于在对记录加独占锁前，先会加上表级别的意向独占锁，那么在加「独占表锁」时，直接查该表是否有意向独占锁，如果有就意味着表里已经有记录被加了独占锁，这样就不用去遍历表里的记录。

所以，意向锁的目的是为了快速判断表里是否有记录被加锁。

- 在使用 InnoDB 引擎的表里对某些记录加上「共享锁」之前，需要先在表级别加上一个「意向共享锁」；
- 在使用 InnoDB 引擎的表里对某些纪录加上「独占锁」之前，需要先在表级别加上一个「意向独占锁」；

意向共享锁和意向独占锁是表级锁，不会和行级的共享锁和独占锁发生冲突，而且意向锁之间也不会发生冲突，只会和共享表锁和独占表锁发生冲突。



**自增锁（AUTO-INC Lock）**

声明 `AUTO_INCREMENT` 属性的字段数据库自动赋递增的值，主要是通过 AUTO-INC 锁实现的。

在插入数据时，会加一个表级别的 AUTO-INC 锁，然后为被 `AUTO_INCREMENT` 修饰的字段赋值递增的值，等插入语句执行完成后，把 AUTO-INC 锁释放掉。

在 MySQL 5.1.22 版本开始，InnoDB 存储引擎提供了一种轻量级的锁来实现自增。一样也是在插入数据的时候，会为被 `AUTO_INCREMENT` 修饰的字段加上轻量级锁，然后给该字段赋值一个自增的值，就把这个轻量级锁释放了，而不需要等待整个插入语句执行完后才释放锁。



### 行级锁

行级锁包括记录锁、间隙锁、临键锁、插入意向锁

在读已提交隔离级别下，行级锁的种类只有记录锁，也就是仅仅把一条记录锁上。

在可重复读隔离级别下，行级锁的种类除了有记录锁，还有间隙锁（目的是为了避免幻读）

**记录锁（Record Lock）**

记录锁，也就是仅仅把一条记录锁上；



**间隙锁（Gap Lock）**

间隙锁，锁定一个范围，但是不包含记录本身；

间隙锁只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象。

间隙锁之间是兼容的，即两个事务可以同时持有包含共同间隙范围的间隙锁，并不存在互斥关系，因为间隙锁的目的是防止插入幻影记录而提出的。



**临键锁（Next-Key Lock）**

临键锁，Record Lock + Gap Lock 的组合，锁定一个范围，并且锁定记录本身，即锁定左开右闭的区间。

如果一个事务获取了 X 型的 next-key lock，那么另外一个事务在获取相同范围的 X 型的 next-key lock 时，是会被阻塞的。数据库默认加临键锁。



**插入意向锁（Insert Intention Lock）**

一个事务在插入一条记录的时候，需要判断插入位置是否已被其他事务加了间隙锁（next-key lock 也包含间隙锁）。

如果有的话，插入操作就会发生阻塞，直到拥有间隙锁的那个事务提交为止（释放间隙锁的时刻），在此期间会生成一个插入意向锁，表明有事务想在某个区间插入新记录，但是现在处于等待状态。



### 怎么加行级锁

>  **什么SQL语句会加行级锁？**

1. 普通的 select 语句是不会对记录加锁的（除了串行化隔离级别），因为它属于快照读，是通过 MVCC（多版本并发控制）实现的。如果要在查询时对记录加行级锁，可以使用下面这两个方式，这两种查询会加锁的语句称为**锁定读**。

   ```sql
   //对读取的记录加共享锁(S型锁)
   select ... lock in share mode;
   
   //对读取的记录加独占锁(X型锁)
   select ... for update;
   ```

   这两条语句必须在一个事务中，因为当事务提交了，锁就会被释放，所以在使用这两条语句的时候，要加上 begin 或者 start transaction 开启事务的语句。

2. update 和 delete 操作都会加行级锁，且锁的类型都是独占锁(X型锁)。



> **行级锁有哪些？**

在读已提交隔离级别下，行级锁的种类只有记录锁，也就是仅仅把一条记录锁上。

在可重复读隔离级别下，行级锁的种类除了有记录锁，还有间隙锁（目的是为了避免幻读），所以行级锁的种类主要有三类：

- Record Lock，记录锁，也就是仅仅把一条记录锁上；
- Gap Lock，间隙锁，锁定一个范围，但是不包含记录本身；
- Next-Key Lock：Record Lock + Gap Lock 的组合，锁定一个范围，并且锁定记录本身。



> **有什么命令可以分析加了什么锁？**

`select * from performance_schema.data_locks\G;`

LOCK_TYPE 中的 RECORD 表示行级锁，而不是记录锁的意思。

LOCK_MODE 可以确认是 next-key 锁，还是间隙锁，还是记录锁：

- 如果 LOCK_MODE 为 `X`，说明是 next-key 锁；
- 如果 LOCK_MODE 为 `X, REC_NOT_GAP`，说明是记录锁；
- 如果 LOCK_MODE 为 `X, GAP`，说明是间隙锁；

![](.\MySql\事务a加锁分析.webp)



**加锁的对象是索引，加锁的基本单位是 next-key lock**，它是由记录锁和间隙锁组合而成的，**next-key lock 是前开后闭区间，而间隙锁是前开后开区间**。

但是，next-key lock 在一些场景下会退化成记录锁或间隙锁。

那到底是什么场景呢？总结一句，**在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成记录锁或间隙锁**。



#### 唯一索引等值查询

当我们用唯一索引进行等值查询的时候，查询的记录存不存在，加锁的规则也会不同：

- 当查询的记录是「存在」的，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会**退化成「记录锁」**。
- 当查询的记录是「不存在」的，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会**退化成「间隙锁」**。



> **为什么唯一索引等值查询并且查询记录存在的场景下，该记录的索引中的 next-key lock 会退化成记录锁？**

原因就是在唯一索引等值查询并且查询记录存在的场景下，仅靠记录锁也能避免幻读的问题。

幻读的定义就是，当一个事务前后两次查询的结果集，不相同时，就认为发生幻读。所以，要避免幻读就是避免结果集某一条记录被其他事务删除，或者有其他事务插入了一条新记录，这样前后两次查询的结果集就不会出现不相同的情况。

- 由于主键具有唯一性，所以**其他事务插入 id = 1 的时候，会因为主键冲突，导致无法插入 id = 1 的新记录**。这样事务 A 在多次查询 id = 1 的记录的时候，不会出现前后两次查询的结果集不同，也就避免了幻读的问题。
- 由于对 id = 1 加了记录锁，**其他事务无法删除该记录**，这样事务 A 在多次查询 id = 1 的记录的时候，不会出现前后两次查询的结果集不同，也就避免了幻读的问题。



> **为什么唯一索引等值查询并且查询记录「不存在」的场景下，在索引树找到第一条大于该查询记录的记录后，要将该记录的索引中的 next-key lock 会退化成「间隙锁」？**

原因就是在唯一索引等值查询并且查询记录不存在的场景下，仅靠间隙锁就能避免幻读的问题。

- 为什么 id = 5 记录上的主键索引的锁不可以是 next-key lock？如果是 next-key lock，就意味着其他事务无法删除 id = 5 这条记录，但是这次的案例是查询 id = 2 的记录，只要保证前后两次查询 id = 2 的结果集相同，就能避免幻读的问题了，所以即使 id =5 被删除，也不会有什么影响，那就没必须加 next-key lock，因此只需要在 id = 5 加间隙锁，避免其他事务插入 id = 2 的新记录就行了。
- 为什么不可以针对不存在的记录加记录锁？锁是加在索引上的，而这个场景下查询的记录是不存在的，自然就没办法锁住这条不存在的记录。



#### 唯一索引范围查询

当唯一索引进行范围查询时，**会对每一个扫描到的索引加 next-key 锁，然后如果遇到下面这些情况，会退化成记录锁或者间隙锁**：

- 情况一：针对「大于等于」的范围查询，因为存在等值查询的条件，那么如果等值查询的记录是存在于表中，那么该记录的索引中的 next-key 锁会**退化成记录锁**。
- 情况二：针对「小于或者小于等于」的范围查询，要看条件值的记录是否存在于表中：
  - 当条件值的记录不在表中，那么不管是「小于」还是「小于等于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。
  - 当条件值的记录在表中，如果是「小于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁；如果「小于等于」条件的范围查询，扫描到终止范围查询的记录时，该记录的索引 next-key 锁不会退化成间隙锁。其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。



#### 非唯一索引等值查询

当我们用非唯一索引进行等值查询的时候，**因为存在两个索引，一个是主键索引，一个是非唯一索引（二级索引），所以在加锁时，同时会对这两个索引都加锁，但是对主键索引加锁的时候，只有满足查询条件的记录才会对它们的主键索引加锁**。

针对非唯一索引等值查询时，查询的记录存不存在，加锁的规则也会不同：

- 当查询的记录「存在」时，由于不是唯一索引，所以肯定存在索引值相同的记录，于是**非唯一索引等值查询的过程是一个扫描的过程，直到扫描到第一个不符合条件的二级索引记录就停止扫描，然后在扫描的过程中，对扫描到的二级索引记录加的是 next-key 锁，而对于第一个不符合条件的二级索引记录，该二级索引的 next-key 锁会退化成间隙锁。同时，在符合查询条件的记录的主键索引上加记录锁**。
- 当查询的记录「不存在」时，**扫描到第一条不符合条件的二级索引记录，该二级索引的 next-key 锁会退化成间隙锁。因为不存在满足查询条件的记录，所以不会对主键索引加锁**。



> **针对非唯一索引等值查询时，查询的值不存在的情况。**

执行 `select * from user where age = 25 for update;` 定位到第一条不符合查询条件的二级索引记录，即扫描到 age = 39，于是**该二级索引的 next-key 锁会退化成间隙锁，范围是 (22, 39)**。

![](.\MySql\非唯一索引等值查询age=25.drawio.webp)

其他事务无法插入 age 值为 23、24、25、26、....、38 这些新记录。不过对于插入 age = 22 和 age = 39 记录的语句，在一些情况是可以成功插入的，而一些情况则无法成功插入。

**插入语句在插入一条记录之前，需要先定位到该记录在 B+树 的位置，如果插入的位置的下一条记录的索引上有间隙锁，才会发生阻塞**。

插入 age = 22 记录的成功和失败的情况分别如下：

- 当其他事务插入一条 age = 22，id = 3 的记录的时候，在二级索引树上定位到插入的位置，而该位置的下一条是 id = 10、age = 22 的记录，该记录的二级索引上没有间隙锁，所以这条插入语句可以执行成功。
- 当其他事务插入一条 age = 22，id = 12 的记录的时候，在二级索引树上定位到插入的位置，而该位置的下一条是 id = 20、age = 39 的记录，正好该记录的二级索引上有间隙锁，所以这条插入语句会被阻塞，无法插入成功。

插入 age = 39 记录的成功和失败的情况分别如下：

- 当其他事务插入一条 age = 39，id = 3 的记录的时候，在二级索引树上定位到插入的位置，而该位置的下一条是 id = 20、age = 39 的记录，正好该记录的二级索引上有间隙锁，所以这条插入语句会被阻塞，无法插入成功。
- 当其他事务插入一条 age = 39，id = 21 的记录的时候，在二级索引树上定位到插入的位置，而该位置的下一条记录不存在，也就没有间隙锁了，所以这条插入语句可以插入成功。

所以，**当有一个事务持有二级索引的间隙锁 (22, 39) 时，插入 age = 22 或者 age = 39 记录的语句是否可以执行成功，关键还要考虑插入记录的主键值，因为「二级索引值（age列）+主键值（id列）」才可以确定插入的位置，确定了插入位置后，就要看插入的位置的下一条记录是否有间隙锁，如果有间隙锁，就会发生阻塞，如果没有间隙锁，则可以插入成功**。



> **针对非唯一索引等值查询时，查询的值存在的情况。**

执行 `select * from user where age = 22 for update;` 

![](.\MySql\非唯一索引等值查询存在.drawio.webp)

在 age = 22 这条记录的二级索引上，加了范围为 (21, 22] 的 next-key 锁，意味着其他事务无法更新或者删除 age = 22 的这一些新记录，不过对于插入 age = 21 和 age = 22 新记录的语句，在一些情况是可以成功插入的，而一些情况则无法成功插入。

- 是否可以插入 age = 21 的新记录，还要看插入的新记录的 id 值，**如果插入 age = 21 新记录的 id 值小于 5，那么就可以插入成功**，因为此时插入的位置的下一条记录是 id = 5，age = 21 的记录，该记录的二级索引上没有间隙锁。**如果插入 age = 21 新记录的 id 值大于 5，那么就无法插入成功**，因为此时插入的位置的下一条记录是 id = 10，age = 22 的记录，该记录的二级索引上有间隙锁。
- 是否可以插入 age = 22 的新记录，还要看插入的新记录的 id 值，从 `LOCK_DATA : 22, 10` 可以得知，其他事务插入 age 值为 22 的新记录时，**如果插入的新记录的 id 值小于 10，那么插入语句会发生阻塞；如果插入的新记录的 id 大于 10，还要看该新记录插入的位置的下一条记录是否有间隙锁，如果没有间隙锁则可以插入成功，如果有间隙锁，则无法插入成功**。

在 age = 39 这条记录的二级索引上，加了范围 (22, 39) 的间隙锁。意味着其他事务无法插入 age 值为 23、24、..... 、38 的这一些新记录。不过对于插入 age = 22 和 age = 39 记录的语句，在一些情况是可以成功插入的，而一些情况则无法成功插入。

- 是否可以插入 age = 22 的新记录，还要看插入的新记录的 id 值，**如果插入 age = 22 新记录的 id 值小于 10，那么插入语句会被阻塞，无法插入**，因为此时插入的位置的下一条记录是 id = 10，age = 22 的记录，该记录的二级索引上有间隙锁（ age = 22 这条记录的二级索引上有 next-key 锁）。**如果插入 age = 22 新记录的 id 值大于 10，也无法插入**，因为此时插入的位置的下一条记录是 id = 20，age = 39 的记录，该记录的二级索引上有间隙锁。
- 是否可以插入 age = 39 的新记录，还要看插入的新记录的 id 值，从 `LOCK_DATA : 39, 20` 可以得知，其他事务插入 age 值为 39 的新记录时，**如果插入的新记录的 id 值小于 20，那么插入语句会发生阻塞，如果插入的新记录的 id 大于 20，则可以插入成功**。



#### 非唯一索引范围查询

**非唯一索引范围查询，索引的 next-key lock 不会有退化为间隙锁和记录锁的情况**，也就是非唯一索引进行范围查询时，对二级索引记录加锁都是加 next-key 锁。

执行 `select * from user where age >= 22  for update;`

![](.\MySql\非唯一索引范围查询age大于等于22.drawio.webp)

是否可以插入age = 21、age = 22 和 age = 39 的新记录，还需要看新记录的 id 值。

> **在 age >= 22 的范围查询中，明明查询 age = 22 的记录存在并且属于等值查询，为什么不会像唯一索引那样，将 age = 22 记录的二级索引上的 next-key 锁退化为记录锁？**

因为 age 字段是非唯一索引，不具有唯一性，所以如果只加记录锁（记录锁无法防止插入，只能防止删除或者修改），就会导致其他事务插入一条 age = 22 的记录，这样前后两次查询的结果集就不相同了，出现了幻读现象。

#### 没有索引的查询

**如果锁定读查询语句，没有使用索引列作为查询条件，或者查询语句没有走索引查询，导致扫描是全表扫描。那么，每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表，这时如果其他事务对该表进行增、删、改操作的时候，都会被阻塞**。

不只是锁定读查询语句不加索引才会导致这种情况，update 和 delete 语句如果查询条件不加索引，那么由于扫描的方式是全表扫描，于是就会对每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表。

因此，**在线上在执行 update、delete、select ... for update 等具有加锁性质的语句，一定要检查语句是否走了索引，如果是全表扫描的话，会对每一个索引加 next-key 锁，相当于把整个表锁住了**，这是挺严重的问题。



### Insert语句怎么加行级锁

Insert 语句在正常执行时是不会生成锁结构的，它是靠聚簇索引记录自带的 trx_id 隐藏列来作为**隐式锁**来保护记录的。

> 什么是隐式锁？

当事务需要加锁的时，如果这个锁不可能发生冲突，InnoDB会跳过加锁环节，这种机制称为隐式锁。隐式锁是 InnoDB 实现的一种延迟加锁机制，其特点是只有在可能发生冲突时才加锁，从而减少了锁的数量，提高了系统整体性能。

隐式锁就是在 Insert 过程中不加锁，只有在特殊情况下，才会将隐式锁转换为显示锁，这里我们列举两个场景。

- 如果记录之间加有间隙锁，为了避免幻读，此时是不能插入记录的；
- 如果 Insert 的记录和已有记录存在唯一键冲突，此时也不能插入记录；

**记录之间加有间隙锁：**

每插入一条新记录，都需要看一下待插入记录的下一条记录上是否已经被加了间隙锁，如果已加间隙锁，此时会生成一个插入意向锁，然后锁的状态设置为等待状态（*PS：MySQL 加锁时，是先生成锁结构，然后设置锁的状态，如果锁状态是等待状态，并不是意味着事务成功获取到了锁，只有当锁状态为正常状态时，才代表事务成功获取到了锁*），现象就是 Insert 语句会被阻塞。

**遇到唯一键冲突：**

如果在插入新记录时，插入了一个与「已有的记录的主键或者唯一二级索引列值相同」的记录（不过可以有多条记录的唯一二级索引列的值同时为NULL，这里不考虑这种情况），此时插入就会失败，然后对于这条记录加上了 **S 型的锁**。

- 如果主键索引重复，插入新记录的事务会给已存在的主键值重复的聚簇索引记录**添加 S 型记录锁**。
- 如果唯一二级索引重复，插入新记录的事务都会给已存在的二级索引列值重复的二级索引记录**添加 S 型 next-key 锁**



> **两个事务执行过程中，执行了相同的 insert 语句的场景**

![](.\MySql\唯一索引加锁.drawio.webp)

两个事务的加锁过程：

- 事务 A 先插入 order_no 为 1006 的记录，可以插入成功，此时对应的唯一二级索引记录被「隐式锁」保护，此时还没有实际的锁结构（执行完这里的时候，你可以看查 performance_schema.data_locks 信息，可以看到这条记录是没有加任何锁的）；
- 接着，事务 B 也插入 order_no 为 1006 的记录，由于事务 A 已经插入 order_no 值为 1006 的记录，所以事务 B 在插入二级索引记录时会遇到重复的唯一二级索引列值，此时事务 B 想获取一个 S 型 next-key 锁，但是事务 A 并未提交，**事务 A 插入的 order_no 值为 1006 的记录上的「隐式锁」会变「显示锁」且锁类型为 X 型的记录锁，所以事务 B 向获取 S 型 next-key 锁时会遇到锁冲突，事务 B 进入阻塞状态**。

并发多个事务的时候，第一个事务插入的记录，并不会加锁，而是会用隐式锁保护唯一二级索引的记录。

但是当第一个事务还未提交的时候，有其他事务插入了与第一个事务相同的记录，第二个事务就会**被阻塞**，**因为此时第一事务插入的记录中的隐式锁会变为显示锁且类型是 X 型的记录锁，而第二个事务是想对该记录加上 S 型的 next-key 锁，X 型与 S 型的锁是冲突的**，所以导致第二个事务会等待，直到第一个事务提交后，释放了锁。



## 日志

### undo log

undo log 是 Innodb 存储引擎层生成的日志，主要用于事务回滚和 MVCC。

`InnoDB`默认是将`Undo-log`存储在`xx.ibdata`共享表数据文件当中，默认采用段的形式存储。

也就是当一个事务尝试写某行表数据时，首先会将旧数据拷贝到`xx.ibdata`文件中，将表中行数据的隐藏字段：`roll_ptr`回滚指针会指向`xx.ibdata`文件中的旧数据，然后再写表上的数据。

那`Undo-log`究竟在`xx.ibdata`文件中怎么存储呢？

在共享表数据文件中，有一块区域名为`Rollback Segment`回滚段，每个回滚段中有`1024`个`Undo-log Segment`，每个`Undo`段可存储一条旧数据，而执行写`SQL`时，`Undo-log`就是写入到这些段中。在`MySQL5.5`版本前，默认只有一个`Rollback Segment`，而在`MySQL5.5`版本后，默认有`128`个回滚段，即支持`128*1024`条`Undo`记录同时存在。

当一个事务需要回滚时，本质上并不会以执行反`SQL`的模式还原数据，而是直接将`roll_ptr`回滚指针指向的`Undo`记录，从`xx.ibdata`共享表数据文件中拷贝到`xx.ibd`表数据文件，覆盖掉原本改动过的数据。



### redo log

redo log是 Innodb 存储引擎层生成的日志，记录当前`SQL`归属事务的状态，以及记录修改内容和修改页的位置。主要用于掉电等故障恢复。

redo log是一种预写式日志，会先记录日志再去更新缓冲区中的数据，所以就算缓冲区数据未被刷写到磁盘，MySQL重启时也可以根据redo log恢复数据，确保持久性。

写的`Redo-log`日志，也是写在内存中的`redo_log_buffer`缓冲区，刷盘策略（`innodb_flush_log_at_trx_commit`控制）：

* 0：有事务提交情况下，每间隔1秒刷写一次日志到磁盘；
* 1：每次提交事务时，都刷写一次日志到磁盘。默认
* 2：每次提交事务时，把日志记录放到内核缓冲区，刷写实际交给操作系统控制。



### bin log

bin log 是 Server 层生成的日志，记录每条`SQL`操作日志，主要是用于数据的主从复制与数据恢复/备份。



> **redo log 和 undo log 区别在哪？**

这两种日志是属于 InnoDB 存储引擎的日志，它们的区别在于：

- redo log 记录了此次事务「**完成后**」的数据状态，记录的是更新**之后**的值；
- undo log 记录了此次事务「**开始前**」的数据状态，记录的是更新**之前**的值；

事务提交之前发生了崩溃，重启后会通过 undo log 回滚事务，事务提交之后发生了崩溃，重启后会通过 redo log 恢复事务



> **redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？**

写入 redo log 的方式使用了追加操作， 所以磁盘操作是**顺序写**，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是**随机写**。

磁盘的「顺序写 」比「随机写」 高效的多，因此 redo log 写入磁盘的开销更小。

可以说这是 WAL 技术的另外一个优点：**MySQL 的写操作从磁盘的「随机写」变成了「顺序写」**，提升语句的执行性能。这是因为 MySQL 的写操作并不是立刻更新到磁盘上，而是先记录在日志上，然后在合适的时间再更新到磁盘上 。

至此， 针对为什么需要 redo log 这个问题我们有两个答案：

- **实现事务的持久性，让 MySQL 有 crash-safe（奔溃恢复） 的能力**，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- **将写操作从「随机写」变成了「顺序写」**，提升 MySQL 写入磁盘的性能。



> **缓存在 redo log buffer 里的 redo log 还是在内存中，它什么时候刷新到磁盘？**

主要有下面几个时机：

- MySQL 正常关闭时；
- 当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间的一半时，会触发落盘；
- InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。
- 每次事务提交时都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘



>**redo log 和 binlog 有什么区别？**

1. 适用对象不同：
   * binlog 是 MySQL 的 Server 层实现的日志，所有存储引擎都可以使用；
   * redo log 是 Innodb 存储引擎实现的日志；

2. 文件格式不同：

   - binlog 有 3 种格式类型，分别是 STATEMENT（默认格式）、ROW、 MIXED，区别如下：
     - STATEMENT：每一条修改数据的 SQL 都会被记录到 binlog 中（相当于记录了逻辑操作，所以针对这种格式， binlog 可以称为逻辑日志），主从复制中 slave 端再根据 SQL 语句重现。但 STATEMENT 有动态函数的问题，比如你用了 uuid 或者 now 这些函数，你在主库上执行的结果并不是你在从库执行的结果，这种随时在变的函数会导致复制的数据不一致；
     - ROW：记录行数据最终被修改成什么样了（这种格式的日志，就不能称为逻辑日志了），不会出现 STATEMENT 下动态函数的问题。但 ROW 的缺点是每行数据的变化结果都会被记录，比如执行批量 update 语句，更新多少行数据就会产生多少条记录，使 binlog 文件过大，而在 STATEMENT 格式下只会记录一个 update 语句而已；
     - MIXED：包含了 STATEMENT 和 ROW 模式，它会根据不同的情况自动使用 ROW 模式和 STATEMENT 模式；

   - redo log 是物理日志，记录的是在某个数据页做了什么修改，比如对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新；

3. 写入方式不同：

   - binlog 是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖以前的日志，保存的是全量的日志。

   - redo log 是循环写，日志空间大小是固定，全部写满就从头开始，保存未被刷入磁盘的脏页日志。

4. 用途不同：

   - binlog 用于备份恢复、主从复制；

   - redo log 用于掉电等故障恢复。



>  **binlog 什么时候刷盘？**

在事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 文件中，但是并没有把数据持久化到磁盘，因为数据还缓存在文件系统的 page cache 里，后续由sync_binlog 参数来控制数据库的 binlog 刷到磁盘上的频率。



### 两阶段提交



>  **为什么需要两阶段提交？**

redo log 影响主库的数据，binlog 影响从库的数据，所以 redo log 和 binlog 必须保持一致才能保证主从数据一致。

MySQL 为了避免出现两份日志之间的逻辑不一致的问题，使用了「两阶段提交」来解决。



> **两阶段提交的过程是怎样的？**

两个阶段提交就是**将 redo log 的写入拆成了两个步骤：prepare 和 commit，中间再穿插写入binlog**，具体如下：

- **prepare 阶段**： 将 redo log 持久化到磁盘（innodb_flush_log_at_trx_commit = 1 的作用），将 redo log 对应的事务状态设置为 prepare；
- **commit 阶段**：将 binlog 持久化到磁盘（sync_binlog = 1 的作用），然后将 redo log 状态设置为 commit。

当在写bin log之前崩溃时：此时 binlog 还没写，redo log 也还没提交，事务会回滚。 日志保持一致 

当在写bin log之后崩溃时： 重启恢复后虽没有commit，但满足prepare和binlog完整，自动commit。日志保持一致 



>  **两阶段提交有什么问题？**

* **磁盘IO次数高**：对于“双1”配置，每个事务提交都会进行两次 fsync（刷盘），一次是 redo log 刷盘，另一次是 binlog 刷盘。
* **锁竞争激烈**：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致。



### 组提交

MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数。

引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个过程：

- **flush 阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
- **sync 阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；
- **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

上面的**每个阶段都有一个队列**，每个阶段有锁进行保护，因此保证了事务写入的顺序，第一个进入队列的事务会成为 leader，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束。



## Buffer Pool





## MyISAM引擎 vs InnoDB引擎

**磁盘文件**

其中使用`MyISAM`引擎的表：`zz_myisam_index`，会在本地生成三个磁盘文件：

- `zz_myisam_index.frm`：该文件中存储表的结构信息。
- `zz_myisam_index.MYD`：该文件中存储表的行数据。
- `zz_myisam_index.MYI`：该文件中存储表的索引数据。

从这里可得知一点：`MyISAM`引擎的表数据和索引数据，会分别放在两个不同的文件中存储。

而反观使用`InnoDB`引擎的表：`zz_innodb_index`，在磁盘中仅有两个文件：

- `zz_innodb_index.frm`：该文件中存储表的结构信息。
- `zz_innodb_index.ibd`：该文件中存储表的行数据和索引数据。

**索引支持**

`MyISAM`表数据和索引数据是分别放在`.MYD、.MYI`文件中，所以注定了`MyISAM`引擎只支持非聚簇索引。而`InnoDB`引擎的表数据、索引数据都放在`.ibd`文件中存储，因此`InnoDB`是支持聚簇索引的。

聚簇索引的要求是：索引键和行数据必须在物理空间上也是连续的，而`MyISAM`表数据和索引数据，分别位于两个磁盘文件中，这也就注定了它无法满足聚簇索引的要求。

**事务机制**

使用`InnoDB`存储引擎的表，可以借助`undo-log`日志实现事务机制。而`MyISAM`并未设计类似的技术，在启动时不会在内存中构建`undo_log_buffer`缓冲区，磁盘中也没有相应的日志文件，因此`MyISAM`并不支持事务机制。

**故障恢复**

`InnoDB`引擎由于`redo-log`日志的存在，因此只要事务提交，机器断电、程序宕机等各种灾难情况，都可以用`redo-log`日志来恢复数据。但`MyISAM`引擎同样没有`redo-log`日志，所以并不支持数据的故障恢复，如果表是使用`MyISAM`引擎创建的，当一条`SQL`将数据写入到了缓冲区后，`SQL`还未被写到`bin-log`日志，此时机器断电、`DB`宕机了，重启之后由于数据在宕机前还未落盘，所以丢了也就无法找回。

**锁粒度**

`MyISAM`仅支持表锁，而`InnoDB`同时支持表锁、行锁。



`MyISAM`引擎优势：

1. `MyISAM`引擎中会记录表的行数，也就是当执行`count()`时，如果表是`MyISAM`引擎，则可以直接获取之前统计的值并返回。`InnoDB`引擎中是不具备的。
2. 当使用`delete`命令清空表数据时，`MyISAM`会直接重新创建表数据文件，而`InnoDB`则是一行行删除数据，因此对于清空表数据的操作，`MyISAM`比`InnoDB`快上无数倍。同时`MyISAM`引擎的表，对于`delete`过的数据不会立即删除，而且先隐藏起来，后续定时删除或手动删除。
3. `MyISAM`引擎中，所有已创建的索引都是非聚簇索引，每个索引之间都是独立的，在索引中存储的是直接指向行数据的地址，而并非聚簇索引的索引键，因此无论走任何索引，都仅需一次即可获得数据，无需做回表查询。



## SQL优化

客户端与连接层的优化：调整客户端`DB`连接池的参数和`DB`连接层的参数。

`MySQL`结构的优化：合理的设计库表结构，表中字段根据业务选择合适的数据类型、索引。一张表最多最多只能允许设计`30`个字段左右，否则会导致查询时的性能明显下降。

`MySQL`参数优化：调整参数的默认值，根据业务将各类参数调整到合适的大小。

整体架构优化：引入中间件减轻数据库压力，优化`MySQL`架构提高可用性。例如redis、MQ、读写分离、分库分表。

编码层优化：根据库表结构、索引结构优化业务`SQL`语句，提高索引命中率。



### 主键优化

1. 满足业务需求的情况下，尽量降低主键的长度；

2. 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键；

3. 尽量不要使用UUID做主键或者是其他自然主键，如身份证号；

4. 业务操作时，避免对主键的修改。

### order by优化

MySQL的排序，有两种方式:

![](MySql\orderby优化.webp)

对于以上的两种排序方式，Using index的性能高，而Using filesort的性能低，我们在优化排序操作时，尽量要优化为 Using index。

order by优化原则：

1. 根据排序字段建立合适的索引，多字段排序时，也遵循最左前缀法则；

2. 尽量使用覆盖索引；

3. 多字段排序, 一个升序一个降序，此时需要注意联合索引在创建时的规则(ASC/DESC)；

4. 如果不可避免的出现filesort，大数据量排序时，可以适当增大排序缓冲区大小sort_buffer_size(默认256k)。

### group by优化

在分组操作中，我们需要通过以下两点进行优化，以提升性能:

1. 在分组操作时，可以通过索引来提高效率；

2. 分组操作时，索引的使用也是满足最左前缀法则的。

### limit优化

在数据量比较大时，如果进行limit分页查询，在查询时，越往后，分页查询效率越低。

因为，当在进行分页查询时，如果执行 limit 2000000,10 ，此时需要MySQL排序前2000010 记录，仅仅返回 2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大 。

优化思路: 一般分页查询时，通过创建覆盖索引能够比较好地提高性能，可以通过覆盖索引加子查询形式进行优化。

```sql
explain select * from tb_sku t , (select id from tb_sku order by id limit 2000000,10) a where t.id = a.id;
```



### count优化

如果数据量很大，在执行count操作时，是非常耗时的。InnoDB 引擎中，它执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。



count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加，最后返回累计值。

![](MySql\count.webp)

性能：

```sql
count(*) = count(1) > count(主键字段) > count(字段)
```

count(1)、 count(*)、 count(主键字段)在执行的时候，如果表里存在二级索引，优化器就会选择二级索引进行扫描。因为二级索引记录比聚簇索引记录占用更少的存储空间。

count(1)时， server 层每从 InnoDB 读取到一条记录，就将 count 变量加 1，不会读取任何字段。

**count(`*`) 其实等于 count(`0`)**，也就是说，当你使用 count(`*`) 时，MySQL 会将 `*` 参数转化为参数 0 来处理。**count(\*) 执行过程跟 count(1) 执行过程基本一样的**

count(字段) 来统计记录个数，它的效率是最差的，会采用全表扫描的方式来统计。



优化思路：

1. 近似值：使用 show table status 或者 explain 命令来表进行估算。
2. 额外表保存计数值（redis或mysql）



### update优化

我们主要需要注意一下update语句执行时的注意事项。

update course set name = 'javaEE' where id = 1 ;

当我们在执行删除的SQL语句时，会锁定id为1这一行的数据，然后事务提交之后，行锁释放。﻿

当我们开启多个事务，再执行如下SQL时：

update course set name = 'SpringBoot' where name = 'PHP' ;

我们发现行锁升级为了表锁。导致该update语句的性能大大降低。

Innodb的行锁是针对索引加的锁，不是针对记录加的锁，并且该索引不能失效，否则会从行锁升级成表锁。



## SQL性能分析

### sql执行频率

Mysql客户端链接成功后，通过以下命令可以查看当前数据库的  insert/update/delete/select  的访问频次：

show [session|global] status like ‘com_____’;

session: 查看当前会话；

global: 查看全局数据；

com insert: 插入次数；

com select: 查询次数；

com delete: 删除次数；

com updat: 更新次数；

通过查看当前数据库是以查询为主，还是以增删改为主，从而为数据库优化提供参考依据，如果以增删改为主，可以考虑不对其进行索引的优化；如果以查询为主，就要考虑对数据库的索引进行优化

### 慢查询日志

慢查询日志记录了所有执行时间超过指定参数（long_query_time,单位秒，默认10秒）的所有sql日志：

开启慢查询日志前，需要在mysql的配置文件中（/etc/my.cnf）配置如下信息：

1. 开启mysql慢日志查询开关：

   ```
   slow_query_log = 1
   ```

   

2. 设置慢日志的时间，假设为2秒，超过2秒就会被视为慢查询，记录慢查询日志：

   ```
   long_query_time=2
   ```

   

3. 配置完毕后，重新启动mysql服务器进行测试:

   ```
   systemctl restarmysqld
   ```

   

4. 查看慢查询日志的系统变量，是否打开：

   ```
   show variables like “slow_query_log”;
   ```

   

5. 查看慢日志文件中（/var/lib/mysql/localhost-slow.log）记录的信息：

   ```
   Tail -f localhost-slow.log
   ```

   

最终发现，在慢查询日志中，只会记录执行时间超过我们预设时间（2秒）的sql，执行较快的sql不会被记录。



### Profile 详情

show profiles 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了。﻿

1. 通过 have_profiling 参数，可以看到mysql是否支持profile 操作：

   ```
   select @@have_profiling;
   ```

   

2. 通过set 语句在session/global 级别开启profiling: 

   ```
   set profiling =1;
   ```

   

​		开关打开后，后续执行的sql语句都会被mysql记录，并记录执行时间消耗到哪儿去了。比如执行以下几条sql语句：﻿

​		select * from tb_user; 

​		select * from tb_user where id = 1; 

​		select * from tb_user where name = '白起'; 

​		select count(*) from tb_sku;



3. 查看每一条sql的耗时基本情况：

   ```
   show profiles;
   ```

   

4. 查看指定的字段的sql 语句各个阶段的耗时情况：

   ```
   show profile for query Query_ID;
   ```

   

5. 查看指定字段的sql语句cpu 的使用情况：

   ```
   show profile cpu for query Query_ID;
   ```

   

### explain 详情

EXPLAIN 或者 DESC命令获取 MySQL 如何执行 SELECT 语句的信息，包括在 SELECT 语句执行过程中，表如何连接和连接的顺序。﻿

`EXPLAIN` 并不会真的去执行相关的语句，而是通过 查询优化器 对语句进行分析，找出最优的查询方案，并显示对应的信息。

语法 :直接在 select 语句之前加上关键字 explain/desc;

![](MySQL\explain.webp)

extra 几个重要的参考指标：

- Using filesort ：当查询语句中包含 group by 操作，而且无法利用索引完成排序操作的时候， 这时不得不选择相应的排序算法进行，甚至可能会通过文件排序，效率是很低的，所以要避免这种问题的出现。
- Using temporary：使了用临时表保存中间结果，MySQL 在对查询结果排序时使用临时表，常见于排序 order by 和分组查询 group by。效率低，要避免这种问题的出现。
- Using index：所需数据只需在索引即可全部获得，不须要再到表中取数据，也就是使用了覆盖索引，避免了回表操作，效率不错。



## 范式

第一范式：确保原子性，表中每一个列数据都必须是不可再分的字段。

第二范式：确保唯一性，每张表都只描述一种业务属性，一张表只描述一件事。

第三范式：确保独立性，表中除主键外，每个字段之间不存在任何依赖，都是独立的。

巴斯范式：主键字段独立性，联合主键字段之间不能存在依赖性。



### 第一范式

所有的字段都是基本数据字段，不可进一步拆分。

### 第二范式

在满足第一范式的基础上，还要满足数据表里的每一条数据记录，都是可唯一标识的。而且所有字段，都必须完全依赖主键，不能只依赖主键的一部分。

把只依赖于主键一部分的字段拆分出去，形成新的数据表。

### 第三范式

在满足第二范式的基础上，不能包含那些可以由非主键字段派生出来的字段，或者说，不能存在依赖于非主键字段的字段。

### 巴斯-科德范式（BCNF）

巴斯-科德范式也被称为`3.5NF`，至于为何不称为第四范式，这主要是由于它是第三范式的补充版，第三范式的要求是：任何非主键字段不能与其他非主键字段间存在依赖关系，也就是要求每个非主键字段之间要具备独立性。而巴斯-科德范式在第三范式的基础上，进一步要求：**任何主属性不能对其他主键子集存在依赖**。也就是规定了联合主键中的某列值，不能与联合主键中的其他列存在依赖关系。



```sh
+-------------------+---------------+--------+------+--------+
| classes           | class_adviser | name   | sex  | height |
+-------------------+---------------+--------+------+--------+
| 计算机-2201班     | 熊竹老师      | 竹子   | 男   | 185cm  |
| 金融-2201班       | 竹熊老师      | 熊猫   | 女   | 170cm  |
| 计算机-2201班     | 熊竹老师      | 子竹   | 男   | 180cm  |
+-------------------+---------------+--------+------+--------+
```

例如这张学生表，此时假设以`classes`班级字段、`class_adviser`班主任字段、`name`学生姓名字段，组合成一个联合主键。在这张表中，一条学生信息中的班主任，取决于学生所在的班级，因此这里需要进一步调整结构：

```sh
SELECT * FROM `zz_classes`;
+------------+-------------------+---------------+
| classes_id | classes_name      | class_adviser |
+------------+-------------------+---------------+
|          1 | 计算机-2201班     | 熊竹老师      |
|          2 | 金融-2201班       | 竹熊老师      |
+------------+-------------------+---------------+

SELECT * FROM `zz_student`;
+------------+--------+------+--------+
| classes_id | name   | sex  | height |
+------------+--------+------+--------+
|          1 | 竹子   | 男   | 185cm  |
|          2 | 熊猫   | 女   | 170cm  |
|          1 | 子竹   | 男   | 180cm  |
+------------+--------+------+--------+
```

经过结构调整后，原本的学生表则又被拆为了班级表、学生表两张，在学生表中只存储班级`ID`，然后使用`classes_id`班级`ID`和`name`学生姓名两个字段作为联合主键。


