# OceanBase的正确使用方法
OceanBase不是设计出来的，而是在使用过程中不断进化出来的。因此，系统使用以及运维的方便性至关重要。

OceanBase的使用者是业务系统开发人员，并交由专门的OceanBase DBA来运维。为了方便业务使用，OceanBase实现了SQL接口且兼容MySQL协议，从而融入到MySQL开源生态圈。MySQL大部分管理工具，例如MySQL客户端，MySQL admin，能够在OceanBase系统中直接使用。另外，OceanBase将系统运维、监控相关的内部信存放到内部的系统表中，从而方便运维、监控系统获取。

OceanBase早期版本只允许通过Java或者C APl接口访问，新版本增加了SQL支持，且兼容MySQL客户端访问协议。OceanBase推荐用户使用SQL，但老应用仍然可以使用以前的JavaAPI访问OceanBase。下面介绍几个访问与使用场景。
## MySQL客户端连接
使用者采用MySQL客户端连接OceanBase。通过MySQL客户端可以查看系统已有的表格、表格schema，执行select、update、insert、delete等SOL语句，查看系统内部状态，以及发送OceanBase集群运维命令。

【例】首先通过create table命令创建一张名称为test的表格，表格包含两列：id和ame，其中id为主键。接着，往表格中写入两行记录（1，“alice”），（2，”tob”）。最后，通过select语句读取这两行数据。
## JDBC访问（JDBC template）
Java 应用通过标准JDBC访问OceanBase，代码如下所示：
```java
ObGroupDataSource groupSource = new OBGroupDataSource();
groupSource.setUserName("user");//设置用户名
groupSource.setPasswd("pass");//设置密码
groupSource.setDbNane("test");// oceanBase不支持db，这里可以填任意值
groupSouorce.setConfigURL(ob_addr_url);//设置OceanBase集群的地址
groupSource.init();//初始化data source
JdbcTemplate jtp = new JdbcTemplate();
jtp.setDatasource(groupsource);//设置 jdbc template依赖的data source
String sql = "select 1 from = dual";
int ret = jtp.queryForInt(sql);//执行SOL查询
```  
## Spring集成
可以通过将OceanBase DataSource集成到Spring中，配置如下：
```xml
<bean id = "groupDataSource" 
        class=”com.alipay.oceanbase,ObGroupDataSource”
        init-method="init">
    <property name="username" value="user" />
    <property name="passwd" value="pass" />
    <property name="dbName" value="test" />
    <property name="configURL" value_ob_addr_url />
</bean>
```  
## C 客户端
C 应用通过OceanBase C客户端访间OceanBase，使用方式与MySQL C客户端完全一致，代码如下：
```sql
MYSQL mysql;
mysql_init(&mysql); //初始化
Mysql_real_connect(&mysql, ob url, ob user, ob pass, NULL, 0, NULL, 0);  //连接OceanBase数据库
Mysql_real_query(&mysq1, sq1, strlen(sql));  //执行SQL查询
MYSQL_RES* res = mysql_store_reault(&mysql); //获取SQL查询结果集
//处理SQL查询返回的结果集
while（MYSQL_ROW row = mysql_fetch_row(res); //从结果集读取一行数据
//处理结果集中的一行结果
Mysql_free_result(res); //释放结果集
Mysql_close(&mysql);  //关闭连接
```  
## 总结
当然，应用可能会在客户端维护OceanBase连接池，Java应用还可能会使用其他持久层框架，例如iBatis。由于OceanBase兼容JDBC和MySQL C客户端，使用MySQL的应用无须修改代码就能接入OceanBase。
