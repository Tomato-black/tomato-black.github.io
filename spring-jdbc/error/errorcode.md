***
Spring-jdbc异常处理
***
>>1、为什么多种数据库异常（数据库不一样Mysql、H2等），最后对应的Spring-jdbc的异常是一样的？

>>2、怎样自定义一个Sql异常？


>>------------------------------------

1、Spring-jdbc Sql异常继承关系
 
      a、org.springframework.dao.DataAccessException
    
      Spring自定义的sql异常基类，Spring里面所有Sql异常都是继承此基类
    
     b、继承关系

2、异常转换
    
     a、SQLExceptionTranslator 
        这是Spring的Sql异常转换接口
        核心方法：protected DataAccessException doTranslate(String task, @Nullable String sql, SQLException ex)
     b、两个重要的实现类
       （1）、SQLErrorCodeSQLExceptionTranslator
          该实现类是将定义好的数据库异常码转换到对应的Spring的异常，因此可以做到准确的异常匹配
       （2）、SQLStateSQLExceptionTranslator
          该实现类是将java.sql.SQLException转换为Spring的异常，因此准确性不是很高，依赖于java.sql.SQLException