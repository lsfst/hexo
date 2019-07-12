---
title: Mybatis配置mysql+sqlite双数据源
date: 2019-1-05
tags: [java,sqlite]
categories: Java
toc: true
---
### 需求
&emsp;&emsp;目前需要对现有业务进行一次拆分，在项目里面配置两个数据源，要求权限，路由相关的数据在sqlite中操作，其他的数据在mysql中操作。sqlite是一款轻型的、嵌入式的关系型数据库，具体使用起来跟mysql大同小异。所以这实际上就是一个多数据源配置的问题，琢磨了好一会，算是找到了一种可行的方案。

### pom.xml
&emsp;&emsp;话不多说，先引入依赖，主要是mybatis和jdbc连接包：

{% codeblock lang:java %}
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.15</version>
</dependency>

<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.21.0.1</version>
</dependency>
{% endcodeblock %}

### 配置

{% codeblock lang:java %}
server.port=9090

spring.datasource.mysql.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.mysql.jdbc-url=jdbc:mysql://localhost:3306/test?serverTimezone=UTC&useSSL=false
spring.datasource.mysql.username=root
spring.datasource.mysql.password=myvifi

spring.datasource.sqlite.driver-class-name = org.sqlite.JDBC
spring.datasource.sqlite.jdbc-url = jdbc:sqlite:sqlite.db
{% endcodeblock %}

其中要注意的点：
{% blockquote %}
1.配置单数据源时数据库配置不能用-隔开，会报错，使用spring.datasource.url 和spring.datasource.driverClassName即可；
2.使用springboot2.0配置多数据源时需要-隔开：如spring.datasource.jdbc-url和spring.datasource.driver-class-name
3.mysql-connector-java 6 以上版本必须指定时区serverTimezone,驱动使用com.mysql.cj.jdbc.Driver
{% endblockquote %}

### 数据源
mysql为主库,sqlite的代码就不贴来，与mysql大体一样，去掉@Primary注解，同时更换对应mapper目录即可

{% codeblock lang:java %}
@Configuration
@MapperScan(basePackages = "com.example.sqliteandmysql.mapper.test1", sqlSessionTemplateRef  = "mysqlSqlSessionTemplate")
public class DataSource1Config {
    @Bean(name = "mysqlDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.mysql")
    @Primary
    public DataSource testDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "mysqlSqlSessionFactory")
    @Primary
    public SqlSessionFactory testSqlSessionFactory( @Qualifier("mysqlDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mybatis/mapper/test1/*.xml"));
        return bean.getObject();
    }

    @Bean(name = "mysqlTransactionManager")
    @Primary
    public DataSourceTransactionManager testTransactionManager( @Qualifier("mysqlDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "mysqlSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate testSqlSessionTemplate( @Qualifier("mysqlSqlSessionFactory") SqlSessionFactory sqlSessionFactory) throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
{% endcodeblock %}

### 项目结构
<div>{% asset_img 1.png 项目结构 %}</div>

### 数据库初始化
#### mysql
{% codeblock %}
DROP TABLE IF EXISTS `users`;
CREATE TABLE `users` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `userName` varchar(32) DEFAULT NULL COMMENT '用户名',
  `passWord` varchar(32) DEFAULT NULL COMMENT '密码',
  `user_sex` varchar(32) DEFAULT NULL,
  `nick_name` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=28 DEFAULT CHARSET=utf8;

INSERT INTO users VALUES(1,"李大姐","654321","WOMAN","李大姐mysql");
{% endcodeblock %}

#### sqlite
{% codeblock %}
DROP TABLE IF EXISTS `users`;
create table users
(
 id int not null,
 passWord char(32),
 userName char(32),
 user_sex char(32),
 nick_name char(32)
);
insert into users values (1,123456,"张三","MAN","张麻子sqlite");

{% endcodeblock %}

顺便说说sqlite中的一些坑：
{% blockquote %}
&emsp;&emsp;1.sqlite自带自增row_id主键，所以不允许自定义自增主键。
&emsp;&emsp;sqlite中每个表都默认包含一个隐藏列rowid(使用WITHOUT ROWID定义的表除外)。通常情况下，rowid可以唯一的标记表中的每个记录。表中插入的第一条记录的rowid为1，后续插入的记录的rowid由当前最大rowid+1得到。但默认的rowid不会持久化，比如：当所有记录被清空时，再插入记录时rowid会重新从1开始计数；也即每次插入数据设置rowid时，都会在当前已有的最大rowid的基础上+1，这一点与mysql的Integer AutoIncreament不同，mysql会跳过已经使用过的自增主键。
&emsp;&emsp;所以如果要实现mysql那样的主键，需要声明WITHOUT ROWI放弃掉原生主键，同时指定新的主键，相当于是给rowid赋予了别名，同时实现了持久化。
&emsp;&emsp;2.模糊类型带来的问题
&emsp;&emsp;由于sqlite的数据类型比较模糊，所以从mysql迁移到sqlite时，javabean中的数据类型会发生微调，比如datetime会转化成String或者Integer，这一点需要在业务中进行处理。
{% endblockquote %}

对应的实体类：
{% codeblock %}
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String userName;
    private String passWord;
    private UserSexEnum userSex;
    private String nickName;
    }
 {% endcodeblock %}
 

### 对外接口
 {% codeblock %}
 @RestController
 public class UserController {
 
 
     @Autowired
     private User1Mapper user1Mapper;
 
     @Autowired
     private User2Mapper user2Mapper;
 
     @RequestMapping("/mysql/getUsers")
     public List< User > get1Users() {
         List<User> users=user1Mapper.getAll();
         return users;
     }
 
     @RequestMapping("/sqlite/getUsers")
     public List< User > get2Users() {
         List<User> users=user2Mapper.getAll();
         return users;
     }
 }
  {% endcodeblock %}
  
  ### 测试结果
  {% asset_img 2.png %}
  
  {% asset_img 3.png %}