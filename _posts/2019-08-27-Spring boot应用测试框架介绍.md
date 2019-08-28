---
layout:     post   				    # 使用的布局（不需要改）
title:      Spring boot应用测试框架介绍				# 标题 
subtitle:   官方提供的测试框架spring-boot-test-starter，虽然提供了很多功能，但是在数据库层面，依旧存在问题，它强烈依赖于数据库中的数据，并且自身不具备数据初始化的能力。 #副标题
date:       2019-08-27				# 时间
author:     凌洛 						# 作者
header-img: img/post-bg-bravo.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - Spring
---

# 一、spring boot应用测试存在的问题

官方提供的测试框架spring-boot-test-starter，虽然提供了很多功能（junit、spring test、assertj、hamcrest、mockito、jsonassert、jsonpath），但是在数据库层面，依旧存在问题，它强烈依赖于数据库中的数据，并且自身不具备数据初始化的能力。测试框架spring-test-dbunit与spring-boot-unitils-starter支持spring-boot应用的测试，同时，也提供单元测试前置数据准备的功能。

# 二、spring-test-dbunit介绍与应用

## 2.1、介绍

spring-test-dbunit是spring boot的作者之一Phillip Webb开发的、用于给spring项目的单元测试提供dbunit功能的开源项目。dbunit项目的介绍为：puts your database into a known state between test runs。spring-test-dbunit的官网介绍为：Spring DBUnit provides integration between the Spring testing framework and the popular DBUnit project。

## 2.2、应用

实例主要代码如下：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = DemoTestApplication.class)
@TestExecutionListeners({
        DependencyInjectionTestExecutionListener.class,
        TransactionDbUnitTestExecutionListener.class})
@DbUnitConfiguration(dataSetLoader = XlsDataSetLoader.class)
@Transactional

public class UserControllerTest {

    @Autowired
    private UserController userController;

    @Test
    @DatabaseSetup({"/data/test_getUsername.xls"})
    public void test_getUsername() {
        String username = userController.getUsername(1234);
        Assert.assertTrue(username.equals("zhangsan"));
    }
}
```
test_getUsername.xls的内容如下：

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/ae68f5e6ccda4c879e260b4f1163c8a746b1abec.png)


## 2.3、实现原理

测试环境准备：

@RunWith(SpringJUnit4ClassRunner.class)：用于启动测试、注册TestExecutionListener、构建testContext。

@SpringBootTest(classes = DemoTestApplication.class)：利用SpringBootTestContextBootstrapper加载applicationContext。

DependencyInjectionTestExecutionListener：用来将bean注入到测试的class中，例如实例中的userController。

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/dab7d870a53cd6c4b70fb38177cf1fbe03153b9d.png)
事务实现原理：

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/de6c2c0f50391129a4b4dde352f7de7723da670c.png)


这里的TransactionalTestExecutionListener简称T,DbUnitTestExecutionListener简称D，如下图：

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/cc57d5172c8c996759da313e6076414b901da5cd.png)

运行流程为：初始化测试类-->开始事务-->准备测试数据-->运行测试方法-->进行expectedData校验-->回滚或者提交事务。这就保证了整个方法的测试过程中，数据准备、测试方法运行、测试数据校验都在一个事务里面，最后事务如果回滚了，就不会在测试数据库中留下测试数据。

# 三、spring-boot-unitils-starter介绍与应用

## 3.1、介绍

unitils框架介绍：Unitils is an open source library aimed at making unit and integration testing easy and maintainable。堪称测试之王，其组成结构如下：

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/909e2e0bb67a4d6e918e092b2754e38d94308950.png)

unitils目前只支持xml配置的spring项目，对于spring-boot项目稍不支持，基于此，我就开源一个项目，用于在unitils和spring-boot应用之间建立起桥梁。

这个开源项目主要做了以下工作：

重写SpringModule->SpringBootModule，支持ApplicationContext的设置ApplicationContext设置到SpringBootModule中DataSource替换支持xls的dataSet
目前可用的版本为：
```java
        <dependency>
            <groupId>com.github.yangjianzhou</groupId>
            <artifactId>spring-boot-unitils-starter</artifactId>
            <version>1.1.0.RELEASE</version>
        </dependency>

```
## 3.2、应用

实例代码如下：

```java
@RunWith(UnitilsBootBlockJUnit4ClassRunner.class)
@SpringBootTest(classes = DemoTestApplication.class)
@Transactional(value = TransactionMode.ROLLBACK)

public class UserControllerTest {

    @SpringBeanByType
    private UserController userController;

    @Test
    @DataSet({"/data/test_getUsername.xls"})
    public void test_getUsername() {
        String username = userController.getUsername(1234);
        Assert.assertTrue(username.equals("zhangsan"));

    }
}
```

## 3.3、实现原理

![image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_content_pic/b6bd2154291456a1cc1729392ed7bd89e97216ac.png)

DatabaseModule下的DatabaseTestListener进行了事务的开启与回滚（提交）。

DbUnitModule下的DbUnitListener进行了dataset的准备与expecteddataset的校验。
SpringBootModule下的SpringTestListener进行了测试类中属性的注入与销毁测试类。

# 四、扩展

[spring-test-dbunit](http://springtestdbunit.github.io/spring-test-dbunit)与[spring-boot-unitils-starter](https://github.com/yangjianzhou/spring-boot-unitils)弥补了spring-boot-test-starter在数据库测试方面的不足，结合框架spring-test-dbunit（或者spring-boot-unitils-starter）与mock工具（mockito）以及一些测试方法，可以很好的完成单元测试。

但是，spring-test-dbunit与spring-boot-unitils-starter各有优缺点，spring-test-dbunit有良好的文档，但是最近更新版本为2016年版，仅仅是数据库层面的测试工具。spring-boot-unitils-starter利用了unitils的优势，可以说是一个测试平台了，虽然说，每年都在发布版本（unitils），但是其文档较少。用户可以根据自己的需要进行选择。

 
附：文中涉及到的测试样例代码：https://github.com/yangjianzhou/spring-boot-test-sample
