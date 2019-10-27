---
title: 'mybatis 动态 sql'
date: 2019-10-28 23:43:40
tags: [Spring Boot]
categories: [Java,Spring]
---

### 前言
假设我们有这样一个表。
```sql
CREATE TABLE `users` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `user_name` varchar(32) DEFAULT NULL COMMENT '用户名',
  `password` varchar(32) DEFAULT NULL COMMENT '密码',
  `user_vip` int(8) DEFAULT 2 COMMENT 'VIP 0不是 1是',
  `user_sex` int(8) DEFAULT 2 COMMENT '0女 1男 2默认',
  `user_age` int(8) DEFAULT 0 COMMENT '年龄',
  `nick_name` varchar(32) DEFAULT NULL COMMENT '昵称',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=28 DEFAULT CHARSET=utf8;
```

当我们使用mybatis查询数据的时候，一般情况，都是类似`select * from tab where id = ?`，这种情况其实比较好处理，但是有些时候我们需要根据不同的情况查询不同的数据库的列，例如上表，我们需要跟进不同的情况来决定是取`user_name`还是`nick_name`，类似`select * from tab where ? = '黎明'`，那么我写的这个句子对吗？

### 搭建mybatis环境
其实不应该讲环境的，这里简单提及一下。我是使用gradle的，maven只会一点点。个人使用的是目前最新的spring boot 2.2.0版本，文档说比之前的版本快了很多，个人测试过，确实快了非常的多。
外层根build.gradle:
```
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${sbv}")
    }
}
allprojects {
    repositories {
        mavenCentral()
    }
}
```

项目build.gradle
```
apply plugin: 'java'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile "org.springframework.boot:spring-boot-starter-web:${sbv}"

    // sqlite
    compile 'org.xerial:sqlite-jdbc:3.21.0'
    // mysql
    compile 'mysql:mysql-connector-java:8.0.17'
    // sql connection
    compile "com.alibaba:druid-spring-boot-starter:1.1.10"
    // mysql helper
    compile "org.mybatis.spring.boot:mybatis-spring-boot-starter:2.0.0"

    testCompile "org.springframework.boot:spring-boot-starter-test:${sbv}"
}
```
这里其实想说一点，开发后端和开发客户端不一样的是，开发后端不需要关注我引入了很多的包是不是占用了很多的空间，都是一下子塞进去的，以我的这个例子来说，其实是用不到这么多依赖的，我们可以使用`./gradlew :data_operate:dependencies`，来查看依赖树，这里的data_operate换成你自己的模块名即可。

### 配置映射
可以参考:[spring-boot-mybatis](https://github.com/ityouknow/spring-boot-examples/blob/master/spring-boot-mybatis/spring-boot-mybatis-xml-mulidatasource)
这里配置Dao到xml文件的映射，这个也是不应该单独拿出来说的，这里说的主要原因是，个人建议用mybatis操作数据库的时候，最好可以使用xml文件来写，因为写起来比较方便，注解的话，到后面写动态sql其实不是特别的方便了。
因为我配置了多数据源所以映射配置放在了java代码里面配置。
```java
@MapperScan(basePackages = ClusterDataSourceConfig.PACKAGE, sqlSessionFactoryRef = "clusterSqlSessionFactory")
public class ClusterDataSourceConfig {

    static final String PACKAGE = "com.dream.qimi.mapper";

    @Value("${cluster.datasource.url}")
    private String url;

    @Value("${cluster.datasource.username}")
    private String user;

    @Value("${cluster.datasource.password}")
    private String password;

    @Value("${cluster.datasource.driverClassName}")
    private String driverClass;

    @Bean(name = "clusterDataSource")
    public DataSource clusterDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(driverClass);
        dataSource.setUrl(url);
        dataSource.setUsername(user);
        dataSource.setPassword(password);
        return dataSource;
    }

    @Bean(name = "clusterTransactionManager")
    public DataSourceTransactionManager clusterTransactionManager() {
        return new DataSourceTransactionManager(clusterDataSource());
    }

    @Bean(name = "clusterSqlSessionFactory")
    public SqlSessionFactory clusterSqlSessionFactory(@Qualifier("clusterDataSource") DataSource clusterDataSource)
            throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(clusterDataSource);
        //这里配置Dao和xml的路径映射
        sessionFactory.setMapperLocations(
                new PathMatchingResourcePatternResolver()
                        .getResources("classpath:mybatis/mapper/*.xml")
        );
        return sessionFactory.getObject();
    }

}
```

### 实际操作
具体可以看官方文档:[动态 SQL](https://mybatis.org/mybatis-3/zh/dynamic-sql.html)或者:[动态 SQL](https://www.w3cschool.cn/mybatis/l5cx1ilz.html)

下面这个是核心:
动态 SQL 通常要做的事情是根据条件包含 where 子句的一部分，mybatis的动态sql还是可以嵌套的。比如：
```
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

mybatis大概支持以下几种表达式:
- if
- choose (when, otherwise)
- trim (where, set)
- foreach

if是一个条件表达语句，但是其实只有if没有else，要表达else可以这样:(当是vip的时候查询女用户，否则查询男用户)
```
<if test="vip==true">
    and user_sex = 0
</if>
<if test="vip==false">
    and user_sex = 1
</if>
```

choose是选择类似switch:(当是vip的时候如果是女性查询18岁的用户，男性查询22的用户，否则查询男用户)
```
<if test="vip==true">
    <choose>
        <when test="user_sex == 0">
            and user_age = 18
        </when>
        <when test="user_sex == 1">
            and user_age = 22
        </when>
        <otherwise></otherwise>
    </choose>
</if>
<if test="vip==false">
    and user_sex = 1
</if>
```
其他自己探索，都是差不多的，意思一看就懂了。