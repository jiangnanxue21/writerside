# SQL

## MySQL

![](Mysql架构图.png)

问题:
1. 如果表 T 中没有字段 k，而你执行了这个语句 select * from T where k=1, 那肯定是会报“不存在这个列”的错误： “Unknown column ‘k’ in ‘where clause’”。你觉得这个错误是在我们上面提到的哪个阶段报出来的呢？

答：分析器

2. 判断一下你对这个表 T 有没有执行查询的权限是在哪个阶段？

答：有些时候，SQL语句要操作的表不只是SQL字面上那些。比如如果有个触发器，得在执行器阶段（过程中）才能确定。优化器阶段前是无能为力的

总结一下在评论中看到的问题的解答
1. 连接器是从权限表里边查询用户权限并保存在一个变量里边以供查询缓存，分析器，执行器在检查权限的时候使用。
2. sql执行过程中可能会有触发器这种在运行时才能确定的过程，分析器工作结束后的precheck是不能对这种运行时涉及到的表进行权限校验的，所以需要在执行器阶段进行权限检查。另外正是因为有precheck这个步骤，才会在报错时报的是用户无权，而不是 k字段不存在（为了不向用户暴露表结构）。
3. 词法分析阶段是从information schema里面获得表的结构信息的。
4. 可以使用连接池的方式，将短连接变为长连接。
5. mysql_reset_connection是mysql为各个编程语言提供的api，不是sql语句。
6. wait_timeout是非交互式连接的空闲超时，interactive_timeout是交互式连接的空闲超时。执行时间不计入空闲时间。这两个超时设置得是否一样要看情况。


### 索引

1. 为什么用B+树

常见模型


| 模型     | 缺点                                                                                      |
|--------|-----------------------------------------------------------------------------------------|
| 哈希表    | 适用于只有等值查询的场景                                                                            |
| 有序数组   | 更新数据的时候，插入一个记录就必须得挪动后面所有的记录，成本太高                                                        |
| 二叉搜索树  | 可能劣化成有序数组                                                                               |
| 平衡二叉树  | 保持这棵树是平衡二叉树，时间复杂度也是 O(log(N))；二叉树是搜索效率最高的，但是实际上大多数的数据库存储却并不使用二叉树。其原因是，索引不止存在内存中，还要写到磁盘上 |
| B-tree | B+树可以更好地利用磁盘预读特性, 更快地进行范围查询                                                             |

100 万节点的平衡二叉树，树高 20。一次查询可能需要访问 20 个数据块。在机械硬盘时代，从磁盘随机读一个数据块需要 10 ms 左右的寻址时间。也就是说，对于一个 100 万行的表，如果使用二叉树来存储，单独访问一个行可能需要 20 个 10 ms 的时间

InnoDB 的一个整数字段索引为例，这个 N 差不多是 1200。这棵树高是 4 的时候，就可以存 1200 的 3 次方个值，这已经 17 亿了。考虑到树根的数据块总是在内存中的，一个 10 亿行的表上一个整数字段的索引，查找一个值最多只需要访问 3 次磁盘。其实，树的第二层也有很大概率在内存中，那么访问磁盘的平均次数就更少了

2. InnoDB 的索引组织结构

![](InnoDB的索引组织结构.png)

- 基于主键索引和普通索引的查询有什么区别？

  如果语句是 select * from T where ID=500，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；

  如果语句是 select * from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为`回表`。

- 在一些建表规范里面要求建表语句里一定要有自增主键。哪些场景下应该使用自增主键，而哪些场景下不应该

  自增主键的插入数据模式，正符合了我们前面提到的递增插入的场景。每次插入一条新记录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂

  典型的KV场景适合用业务字段直接做主键

- 覆盖索引

回表的问题，如何解决？

```SQL
// 需要执行几次树的搜索操作，会扫描多少行？
mysql> select * from T where k between 3 and 5
```
在这个过程中，回到主键索引树搜索的过程，我们称为回表。这个查询过程读了k索引树的3条记录，回表了两次

1. 在k索引树上找到 k=3 的记录，取得 ID = 300； -- 查找k
2. 再到ID索引树查到 ID=300 对应的 R3； -- 回表
3. 在k索引树取下一个值 k=5，取得 ID=500； -- 查找k
4. 再回到 ID 索引树查到 ID=500 对应的 R4；-- 回表
5. 在k索引树取下一个值 k=6，不满足条件，循环结束。 -- 查找k

建立联合索引(ID, k)

如果执行的语句是 select ID from T where k between 3 and 5，这时只需要查 ID 的值，而 ID 的值已经在 k 索引树上了，因此可以直接提供查询结果，不需要回表。也就是说，在这个查询里面，索引 k 已经“覆盖了”我们的查询需求，我们称为覆盖索引。

- 最左前缀原则

在建立联合索引的时候，如何安排索引内的字段顺序?

评估标准是，索引的复用能力。因为可以支持最左前缀，所以当已经有了 (a,b) 这个联合索引后，一般就不需要单独在 a 上建立索引了。因此，第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的

- 索引下推

不符合最左前缀的部分，会怎么样呢？模糊查询阻断了age索引的使用

```SQL
// 以市民表的联合索引（name, age）为例
mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
```

优化之前 5.6之前：

![](无索引下推执行流程.png)


可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

![](索引下推执行流程.png)


#### 索引相关问题






## Mybatis

### BaseExecutor

BaseExecutor是模板方法，子类只要实现四个基本方法doUpdate，doFlushStatements，doQuery，doQueryCursor

![](mysql执行过程.png)


```

# 1. SimpleExecutor
  @Test
  void testSimpleExecutor() throws Exception {
    SimpleExecutor executor = new SimpleExecutor(config, new JdbcTransaction(ds, null, false));
    MappedStatement selectStatement = ExecutorTestHelper.prepareSelectOneAuthorMappedStatement(config);
    Author author = new Author(-1, "someone", "******", "someone@apache.org", null, Section.NEWS);
    executor.doQuery(selectStatement, author.getId(), RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, selectStatement.getBoundSql(author.getId()));
    executor.doQuery(selectStatement, author.getId(), RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, selectStatement.getBoundSql(author.getId()));
  }

  output:
    DEBUG [main] - Opening JDBC Connection
    DEBUG [main] - Setting autocommit to false on JDBC Connection [org.apache.derby.impl.jdbc.EmbedConnection@1590481849 (XID = 1140), (SESSIONID = 5), (DATABASE = ibderby), (DRDAID = null) ]
    DEBUG [main] - ==>  Preparing: SELECT * FROM author WHERE id = ?
    DEBUG [main] - ==> Parameters: -1(Integer)
    DEBUG [main] - <==      Total: 0
    DEBUG [main] - ==>  Preparing: SELECT * FROM author WHERE id = ?
    DEBUG [main] - ==> Parameters: -1(Integer)
    DEBUG [main] - <==      Total: 0

# 2. ReuseExecutor 预处理只执行了一次
ReuseExecutor executor = new ReuseExecutor(config, new JdbcTransaction(ds, null, false));
 output:
 DEBUG [main] - Opening JDBC Connection
DEBUG [main] - Setting autocommit to false on JDBC Connection [org.apache.derby.impl.jdbc.EmbedConnection@1777043124 (XID = 1317), (SESSIONID = 5), (DATABASE = ibderby), (DRDAID = null) ]
DEBUG [main] - ==>  Preparing: SELECT * FROM author WHERE id = ?
DEBUG [main] - ==> Parameters: -1(Integer)
DEBUG [main] - <==      Total: 0
DEBUG [main] - ==> Parameters: -1(Integer)
DEBUG [main] - <==      Total: 0


# 3. BatchExecutor 大量的修改操作, 查询操作和simple没有区别
    BatchExecutor executor = new BatchExecutor(config, new JdbcTransaction(ds, null, false));

```

#### 一级缓存

一级缓存和获取连接在公共部分，所有放在BaseExecutor

```
 try (SqlSession sqlSession1 = sqlSessionFactory.openSession()) {
      PersonMapper pm = sqlSession1.getMapper(PersonMapper.class);
      List<Person> people = pm.selectById(1); // selectById是statementID
//      sqlSession1.clearCache();
      List<Person> people1 = pm.selectById(1);
      System.out.println(people1 == people);
```

命中一级缓存情况
1. sql和参数必须相同
2. 必须相同的statementID
3. sqlSession必须一样 --- `会话级缓存`
4. RowBounds必须相同

一级缓存失效：
1. 手动清空 -- sqlSession1.clearCache
2. @Options(flushCache = FlushCachePolicy.TRUE)
3. 执行update
4. 缓存作用域缩小到STATEMENT --修改配置文件/mybatis-config.xml
```
public enum LocalCacheScope {
  SESSION,STATEMENT
}

```
```
<configuration>
    <settings>
        <setting name="defaultExecutorType" value="SIMPLE"/>
        <setting name="useGeneratedKeys" value="true"/>
        <setting name="localCacheScope" value="STATEMENT"/>
    </settings>

```

<seealso>
    <category ref="wrs">
        <a href="https://plugins.jetbrains.com/plugin/20158-writerside/docs/markup-reference.html">Markup reference</a>
        <a href="https://plugins.jetbrains.com/plugin/20158-writerside/docs/manage-table-of-contents.html">Reorder topics in the TOC</a>
        <a href="https://plugins.jetbrains.com/plugin/20158-writerside/docs/local-build.html">Build and publish</a>
        <a href="https://plugins.jetbrains.com/plugin/20158-writerside/docs/configure-search.html">Configure Search</a>
    </category>
</seealso>


Netty特性:
1. 异步事件驱动( asynchronous event-driven )
2. 可维护性(maintainable)
3. 高性能协议服务器和客户端构建(high performance protocol servers & clients)


专题1： 字符串匹配，高亮
专题2： FST，向量
专题3： mapreduce
专题4： mq的幂等性等等
专题5： java类的加载，spi, lucene用到的spi


第3章 k近邻法
指示函数I是定义在某集合X上的函数，表示其中有哪些元素属于某一子集A
argmax argmax(f(x))是使得 f(x)取得最大值所对应的变量点x(或x的集合)
近似误差（approximation error）: 对现有训练集的训练误差
估计误差（estimation error）: 数据处理过程中对误差的估计
交叉验证: 常并不会把所有的数据集都拿来训练，而是分出一部分来（这一部分不参加训练）对训练集生成的参数进行测试，相对客观的判断这些参数对训练集之外的数据的符合程度。这种思想就称为交叉验证（Cross Validation）

三大要素：
距离度量（如欧氏距离）、k值及分类决策规则（如多数表决）

k近邻法的实现：kd树

演进过程
前缀树 -》 FSM -> FSA -> FST


IO的几个实验
1. client没有连接，数据在内核里保存
2. tcp的几个参数buffer, nodelay, keepalive

同步异步 和 阻塞和非阻塞区别  lsof

a^b  ---无进位相加信息
(a&b)<<1 ---进位相加信息
a-b = a + (-b) = a + (~b + 1)

乘法：
a: 0110
b: 1110

a * b 
1) a<<0, b >> 0
2) a<<1, b >> 1

List类支持对列表头部的快速访问，而对尾部访问则没那么高效