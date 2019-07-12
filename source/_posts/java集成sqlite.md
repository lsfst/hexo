---
title: java集成sqlite
date: 2018-12-23
tags: [java,sqlite]
categories: Java
toc: true
---
#### sqlite是啥？
&emsp;&emsp;sqlite，是一款轻型的、嵌入式的关系型数据库。它占用资源非常的低，在嵌入式设备中，可能只需要几百K的内存就够了。它能够支持Windows/Linux/Unix等等主流的操作系统，同时能够跟很多程序语言相结合。与Mysql、PostgreSQL相比，sqlite的处理速度更快。它的使用非常简单，无需安装和管理。一个完整的 sqlite 数据库是存储在一个单一的跨平台的磁盘文件，简单的说一个数据库就是一个单一文件。SQLite虽然很小巧，但是支持的SQL语句不会逊色于其他开源数据库，它支持的大多数的SQL语句。目前最新的版本是sqlite3。

#### 安装流程
[下载地址](https://www.sqlite.org/download.html)

##### windows
{% blockquote %}
1.下载 sqlite-tools-win32-*.zip 和 sqlite-dll-win32-*.zip 压缩文件
2.解压上面两个压缩文件，将得到 sqlite3.def、sqlite3.dll 和 sqlite3.exe 文件
3添加 PATH 环境变量
{% endblockquote %}

##### linux/MacOS
&emsp;&emsp;目前linux 操作系统都附带 sqlite，可以使用命令：sqlite3 来检查机器上是否已经安装了 SQLite。
&emsp;&emsp;万一没安装，可以下载sqlite-autoconf-*.tar.gz解压安装
{% codeblock %}
$tar xvfz sqlite-autoconf-3071502.tar.gz
$cd sqlite-autoconf-3071502
$./configure --prefix=/usr/local
$make
$make install
{% endcodeblock %}

#### Java集成sqlite
&emsp;&emsp;相对于mysql来说，在项目中使用sqlite的好处在于可以省去外部mysql的配置安装，直接将sqlite数据库的单一文件放在项目里一起打包过去，或者直接指定机器磁盘上的某个位置生成数据库文件。
在java项目中内嵌sqlite很简单，无需手动安装数据库。对于springboot项目，只需要引入sqlite-jdbc 和 spring-boot-starter-jdbc依赖即可。
{% codeblock %}
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.21.0.1</version>
</dependency>
{% endcodeblock %}

简单封装的数据库工具类：

{% codeblock  lang:java  %}
public class SqliteUtils {
    final static Logger logger = LoggerFactory.getLogger( SqliteUtils.class);


    private String dbFilePath;
    private Connection connection;
    private Statement statement;
    private ResultSet resultSet;

    /**
     * 构造函数
     * @throws ClassNotFoundException
     * @throws SQLException
     */
    public SqliteUtils(String dbFilePath) throws ClassNotFoundException, SQLException {
        this.dbFilePath=dbFilePath;
        connection = getConnection(dbFilePath);
        logger.info( dbFilePath+" connected successful");
    }

    /**
     * 获取数据库连接
     * @param dbFilePath db文件路径
     * @return 数据库连接
     * @throws ClassNotFoundException
     * @throws SQLException
     */
    public Connection getConnection(String dbFilePath) throws ClassNotFoundException, SQLException {
        //sqlite在进行连接时，若找不到数据库，会自动在对应目录创建一个db文件。
        Connection conn = null;
        Class.forName("org.sqlite.JDBC");
        conn = DriverManager.getConnection("jdbc:sqlite:" + dbFilePath);
        return conn;
    }

    /**
     * 执行sql查询
     * @param sql sql select 语句
     * @param rse 结果集处理类对象
     * @return 查询结果
     * @throws SQLException
     * @throws ClassNotFoundException
     */
    public <T> T executeQuery(String sql, ResultSetExtractor<T> rse) throws SQLException, ClassNotFoundException {
        try {
            resultSet = getStatement().executeQuery(sql);
            T rs = rse.extractData(resultSet);
            return rs;
        } finally {
            destroyed();
        }
    }

    /**
     * 执行select查询，返回结果列表
     *
     * @param sql sql select 语句  
     * @param rm 结果集的行数据处理类对象
     * @return
     * @throws SQLException
     * @throws ClassNotFoundException
     */
    public <T> List<T> executeQuery( String sql, RowMapper<T> rm) throws SQLException, ClassNotFoundException {
        List<T> rsList = new ArrayList<T>();
        try {
            resultSet = getStatement().executeQuery(sql);
            while (resultSet.next()) {
                rsList.add(rm.mapRow(resultSet, resultSet.getRow()));
            }
        } finally {
            destroyed();
        }
        return rsList;
    }

    /**
     * 执行数据库更新sql语句
     * @param sql
     * @return 更新行数
     * @throws SQLException
     * @throws ClassNotFoundException
     */
    public int executeUpdate(String sql) throws SQLException, ClassNotFoundException {
        try {
            int c = getStatement().executeUpdate(sql);
            return c;
        } finally {
            destroyed();
        }

    }

    /**
     * 执行多个sql更新语句
     * @param sqls
     * @throws SQLException
     * @throws ClassNotFoundException
     */
    public void executeUpdate(String...sqls) throws SQLException, ClassNotFoundException {
        try {
            for (String sql : sqls) {
                getStatement().executeUpdate(sql);
            }
        } finally {
            destroyed();
        }
    }

    /**
     * 执行数据库更新 sql List
     * @param sqls sql列表
     * @throws SQLException
     * @throws ClassNotFoundException
     */
    public void executeUpdate(List<String> sqls) throws SQLException, ClassNotFoundException {
        try {
            for (String sql : sqls) {
                getStatement().executeUpdate(sql);
            }
        } finally {
            destroyed();
        }
    }

    private Connection getConnection() throws ClassNotFoundException, SQLException {
        if (null == connection) connection = getConnection(dbFilePath);
        return connection;
    }

    private Statement getStatement() throws SQLException, ClassNotFoundException {
        if (null == statement) statement = getConnection().createStatement();
        return statement;
    }

    /**
     * 数据库资源关闭和释放：rs->st->conn 按照顺序关闭
     */
    public void destroyed() {
        try {
            if (null != resultSet) {
                resultSet.close();
                resultSet = null;
            }

            if (null != statement) {
                statement.close();
                statement = null;
            }

            if (null != connection) {
                connection.close();
                connection = null;
            }

        } catch (SQLException e) {
            logger.error("Sqlite 数据库关闭时异常", e);
        }
    }
}
{% endcodeblock %}

#### 连接测试
{% codeblock lang:java  %}
@Test
public void testHelper() {
    try {
        SqliteUtils h = new SqliteUtils("sqlite.db");
        h.executeUpdate("drop table if exists user;");
        h.executeUpdate("create table user(name varchar(20),desc TEXT);");
        h.executeUpdate("insert into user (name) values('zhangsan');");
        h.executeUpdate("insert into user (name) values('lisi');");
        h.executeUpdate("insert into user (name) values('wangwu');");
        List<String> sList = h.executeQuery("select name from user", new RowMapper<String>() {
            @Override
            public String mapRow( ResultSet rs, int index)
                    throws SQLException {
                return rs.getString("name");
            }
        });
        for(String s:sList){
            System.out.println(s);
        }

    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (SQLException e) {
        e.printStackTrace();
    }
}
{% endcodeblock %}

&emsp;&emsp;连接的时候首先会去检测是否存在数据库文件，不存在则自动创建sqlite.db文件，所以无需手动创建db文件。


