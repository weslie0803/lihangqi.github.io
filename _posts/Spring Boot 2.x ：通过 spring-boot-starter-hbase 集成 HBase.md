# 本文内容
- HBase 简介和应用场景
- spring-boot-starter-hbase 开源简介
- 集成 HBase 实战
- 小结

# 一、HBase 简介和应用场景
## 1.1 HBase 是什么？
HBase 是在 Hadoop 分布式文件系统（简称：HDFS）之上的分布式面向列的数据库。而且是 2007 最初原型，历史悠久。
那追根究底，Hadoop 是什么？Hadoop是一个分布式环境存储并处理大数据。Hadoop 使用 MapReduce 算法统计分析大数据。这时候不得不说下 Google 的著名的三篇大数据的论文，分别讲述 GFS、MapReduce、BigTable，详见[ https://www.bysocket.com/archives/2051](https://www.bysocket.com/archives/2051)。
那回到 HBase，HBase 在 Hadoop 之上提供了类似 BigTable 的能力，它不同于一般的关系数据库，是一个适合非结构化数据存储的数据库。它也不同于行式数据库，是基于列的模式。

HBase 一个面向列的数据库，排序由行决定。简而言之:

- 表是行的集合。
- 行是列族的集合。列族，就是键值对。每个列族以 key 为列命名，可以有无数的列。
- 列族就是列的集合。列连续存储，并且每个单元会有对应的时间戳
- 列的存储也是键值对。
与行式数据库最大的区别就是，可以面向列设计巨大表，适用于在线分析处理 OLAP。
与关系型数据库 RDBMS 也有些区别如下：
![图表](https://github.com/lihangqi/My-blog/blob/master/picture/d9831dcb473c3f87d7336893ff81a3904688c7d4.jpeg)

- HBase 宽表，横向扩展。RDBMS 小表，难成规模
- HBase 没有事务
- HBase 无规范化数据，都是键值对 key value
## 1.2 HBase 应用场景
官网上 hbase.apache.org，特性这么多：
> Features：
>Linear and modular scalability.
>Strictly consistent reads and writes.
>Automatic and configurable sharding of tables
>Automatic failover support between RegionServers.
>Convenient base classes for backing Hadoop MapReduce jobs with Apache HBase tables.
>Easy to use Java API for client access.
>Block cache and Bloom Filters for real-time queries.
>Query predicate push down via server side Filters
>Thrift gateway and a REST-ful Web service that supports XML, Protobuf, and binary data encoding options
>Extensible jruby-based (JIRB) shell
>Support for exporting metrics via the Hadoop metrics subsystem to files or Ganglia; or via JMX

最主要的还是特性能有什么应用场景？大致搜集了下业界的：
- 监控数据的日志详情
- 交易订单的详情数据（淘宝、有赞）
- facebook 的消息详情
# 二、spring-boot-starter-hbase 开源简介
spring-boot-starter-hbase 是自定义的spring-boot 的 hbase starter，为 hbase 的 query 和更新等操作提供简易的 api 并集成spring-boot 的 auto configuration。

具体地址：[https://github.com/SpringForAll/spring-boot-starter-hbase](https://github.com/SpringForAll/spring-boot-starter-hbase)

# 三、集成 HBase 实战
具体代码地址：[https://github.com/JeffLi1993/springboot-learning-example](https://github.com/JeffLi1993/springboot-learning-example)

工程名：springboot-hbase

## 3.1 安装 spring-boot-starter-hbase 组件依赖

因为不在公共仓库，只能自行安装。如果有 maven 私库，可以考虑安装到私库。
下载项目到本地：
```shell
git clone https://github.com/SpringForAll/spring-boot-starter-hbase.git
```  
安装依赖：
```shell
cd spring-boot-starter-hbase
mvn clean install
```  
等待安装完毕即可。

## 3.2 工程集成依赖
目录结构如下：
```shell
springboot-hbase git:(master)
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── spring
        │           └── springboot
        │               ├── Application.java
        │               ├── controller
        │               │   └── CityRestController.java
        │               ├── dao
        │               │   └── CityRowMapper.java
        │               ├── domain
        │               │   └── City.java
        │               └── service
        │                   ├── CityService.java
        │                   └── impl
        │                       └── CityServiceImpl.java
        └── resources
            └── application.properties
```  
先在 pom.xml 加入 spring-boot-starter-hbase 组件依赖，也就是上面安装的依赖，核心加入代码如下：
```java
<properties>
        <hbase-spring-boot>1.0.0.RELEASE</hbase-spring-boot>
    </properties>

    <!-- Spring Boot HBase 依赖 -->
        <dependency>
            <groupId>com.spring4all</groupId>
            <artifactId>spring-boot-starter-hbase</artifactId>
            <version>${hbase-spring-boot}</version>
        </dependency>
```  
然后配置相关 HBase 连接信息，具体 HBase 安装，网上文章一大堆。在 spring-boot 项目的 application.properties 文件中加入对应的配置项目，并检查配置是否正确：
```shell
## HBase 配置
spring.data.hbase.quorum=xxx
spring.data.hbase.rootDir=xxx
spring.data.hbase.nodeParent=xxx
```  
具体配置项信息如下：

- spring.data.hbase.quorum 指定 HBase 的 zk 地址
- spring.data.hbase.rootDir 指定 HBase 在 HDFS 上存储的路径
- spring.data.hbase.nodeParent 指定 ZK 中 HBase 的根 ZNode
## 3.3 HBase 保存查询操作
定义 DTO ，即 domain 包下的 City 对象：
```java
public class City {

    /**
     * 城市编号
     */
    private Long id;

    /**
     * 省份年龄
     */
    private Integer age;

    /**
     * 城市名称
     */
    private String cityName;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getCityName() {
        return cityName;
    }

    public void setCityName(String cityName) {
        this.cityName = cityName;
    }
}
```  
然后定义该对象的 RowMapper，是用来和 HBase 存储作为映射：
```java
public class CityRowMapper implements RowMapper<City> {

    private static byte[] COLUMN_FAMILY = "f".getBytes();
    private static byte[] NAME = "name".getBytes();
    private static byte[] AGE = "age".getBytes();

    @Override
    public City mapRow(Result result, int rowNum) throws Exception {
        String name = Bytes.toString(result.getValue(COLUMN_FAMILY, NAME));
        int age = Bytes.toInt(result.getValue(COLUMN_FAMILY, AGE));

        City dto = new City();
        dto.setCityName(name);
        dto.setAge(age);
        return dto;
    }
}
```  
然后可以用 spring-boot-starter-hbase 组件的 HbaseTemplate 操作 HBase API 。具体操作逻辑写在 CityServiceImpl 业务逻辑实现：
```java
@Service
public class CityServiceImpl implements CityService {

    @Autowired private HbaseTemplate hbaseTemplate;

    public List<City> query(String startRow, String stopRow) {
        Scan scan = new Scan(Bytes.toBytes(startRow), Bytes.toBytes(stopRow));
        scan.setCaching(5000);
        List<City> dtos = this.hbaseTemplate.find("people_table", scan, new CityRowMapper());
        return dtos;
    }

    public City query(String row) {
        City dto = this.hbaseTemplate.get("people_table", row, new CityRowMapper());
        return dto;
    }

    public void saveOrUpdate() {
        List<Mutation> saveOrUpdates = new ArrayList<Mutation>();
        Put            put           = new Put(Bytes.toBytes("135xxxxxx"));
        put.addColumn(Bytes.toBytes("people"), Bytes.toBytes("name"), Bytes.toBytes("test"));
        saveOrUpdates.add(put);

        this.hbaseTemplate.saveOrUpdates("people_table", saveOrUpdates);
    }
}
```  
HbaseTemplate 提供常见的操作接口如下：

- HbaseTemplate.find 返回 HBase 映射的 City 列表
- HbaseTemplate.get 返回 row 对应的 City 信息
- HbaseTemplate.saveOrUpdates 保存或者更新
如果 HbaseTemplate 操作不满足需求，完全可以使用 hbaseTemplate 的getConnection() 方法，获取连接。进而类似 HbaseTemplate 实现的逻辑，实现更复杂的需求查询等功能

具体代码地址：[https://github.com/JeffLi1993/springboot-learning-example](https://github.com/JeffLi1993/springboot-learning-example)

工程名：springboot-hbase

# 四、小结
其实 starter 这种好处，大家也都知道。低耦合高内聚，类似 JDBCTemplate，将操作 HBase、ES 也好的 Client 封装下。然后每个业务工程拿来即用，不然肯定会有重复代码出现。
另外还是强调一点，合适的业务场景选择 HBase，常见如下:
- 监控数据的日志详情
- 交易订单的详情数据（淘宝、有赞）
- facebook 的消息详情
