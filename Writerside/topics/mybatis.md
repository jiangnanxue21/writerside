# Mybatis

## BaseExecutor
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

### 一级缓存

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
3. sqlSession必须一样 --- 会话级缓存
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