---
title: sqlite相关介绍
date: 2018-12-23
tags: [sqlite]
categories: Java
toc: true
---
### 数据类型
&emsp;&emsp;就大部分的增删查改操作而言，sqlite与mysql没多大区别。不过，与mysql不同的是，sqlite的数据类型是动态数据类型，值的数据类型和值本身，而不是和它的容器，关联在一起的。
#### 存储类
&emsp;&emsp;每个存储在SQLite数据库中（或被数据库引擎操纵的）的值都有下列存储类的一个：

{% blockquote %}
1.NULL：空值。 
2.INTEGER：带符号的整型，具体取决于存入数字的范围大小。 
3.REAL：浮点数字，存储为8-byte IEEE浮点数。 
4.TEXT：字符串文本。 
5.BLOB：二进制对象。 
{% endblockquote %}

#### 数据类型
&emsp;&emsp;实际上，sqlite3也接受如下的数据类型：

{% blockquote %} 
smallint 16 位元的整数。 
integer 32 位元的整数。 
decimal(p,s) p 精确值和 s 大小的十进位整数，精确值p是指全部有几个数(digits)大小值，s是指小数点後有几位数。如果没有特别指定，则系统会设为 p=5; s=0 。 
float  32位元的实数。 
double  64位元的实数。 
char(n)  n 长度的字串，n不能超过 254。 
varchar(n) 长度不固定且其最大长度为 n 的字串，n不能超过 4000。 
graphic(n) 和 char(n) 一样，不过其单位是两个字元 double-bytes， n不能超过127。这个形态是为了支援两个字元长度的字体，例如中文字。 
vargraphic(n) 可变长度且其最大长度为 n 的双字元字串，n不能超过 2000 
date  包含了 年份、月份、日期。 
time  包含了 小时、分钟、秒。 
timestamp 包含了 年、月、日、时、分、秒、千分之一秒。 
datetime 包含日期时间格式，必须写成'2010-08-05'不能写为'2010-8-5'，否则在读取时会产生错误！ 
{% endblockquote %}

**&emsp;&emsp;注意存储类（storage class）比数据类型更一般。INTEGER存储类，例如，包含6种长度不同的整数数据类型。这在磁盘中是有区别的。不过一旦INTEGER值从磁盘读到内容中进行处理的时候，这些值会转化为更普通的数据类型（8位有符号整数）。因此在大部分情况下，存储类和数据类型是不易分辨的，这两个术语可以交换使用。**

&emsp;&emsp;当然，由于SQLite是无类型的. 所以可以保存任何类型的数据到任何表的任何列中, 无论这列声明的数据类型是什么(只有自动递增Integer Primary Key才有用). 对于SQLite来说对字段不指定类型是完全有效的. 如: 

{% codeblock %}
Create Table ex3(a, b, c); 
{% endcodeblock %}

&emsp;&emsp;尽管SQLite允许忽略数据类型, 最好还是在创建表时指定数据类型. 因为指定数据类型便于理解交流, 也便于更换数据库引擎。常见的数据类型有：

{% codeblock %}
CREATE TABLE IF NOT EXISTS ex2(      
a VARCHAR(10),      
b NVARCHAR(15),     
c TEXT,      
d INTEGER,     
e FLOAT,     
f BOOLEAN,      
g CLOB,      
h BLOB,      
i TIMESTAMP,     
j NUMERIC(10,5),      
k VARYING CHARACTER (24),      
l NATIONAL VARYING CHARACTER(16)     
); 
{% endcodeblock %}

#### char、varchar、text和nchar、nvarchar、ntext的区别 
{% blockquote %} 
1、CHAR。CHAR存储定长数据很方便，CHAR字段上的索引效率级高，比如定义char(10)，那么不论你存储的数据是否达到了10个字节，都要占去10个字节的空间,不足的自动用空格填充。 

2、VARCHAR。存储变长数据，但存储效率没有CHAR高。如果一个字段可能的值是不固定长度的，我们只知道它不可能超过10个字符，把它定义为 VARCHAR(10)是最合算的。VARCHAR类型的实际长度是它的值的实际长度+1。为什么“+1”呢？这一个字节用于保存实际使用了多大的长度。从空间上考虑，用varchar合适；从效率上考虑，用char合适，关键是根据实际情况找到权衡点。 

3、TEXT。text存储可变长度的非Unicode数据，最大长度为2^31-1(2,147,483,647)个字符。 

4、NCHAR、NVARCHAR、NTEXT。这三种从名字上看比前面三种多了个“N”。它表示存储的是Unicode数据类型的字符。我们知道字符中，英文字符只需要一个字节存储就足够了，但汉字众多，需要两个字节存储，英文与汉字同时存在时容易造成混乱，Unicode字符集就是为了解决字符集这种不兼容的问题而产生的，它所有的字符都用两个字节表示，即英文字符也是用两个字节表示。nchar、nvarchar的长度是在1到4000之间。和char、varchar比较起来，nchar、nvarchar则最多存储4000个字符，不论是英文还是汉字；而char、varchar最多能存储8000个英文，4000个汉字。可以看出使用nchar、nvarchar数据类型时不用担心输入的字符是英文还是汉字，较为方便，但在存储英文时数量上有些损失。  
{% endblockquote %}

&emsp;&emsp;所以一般来说，如果含有中文字符，用nchar/nvarchar，如果纯英文和数字，用char/varchar。 

&emsp;&emsp;SQL语句中的所有值，不管是SQL语句中嵌入的字面值，还是预编译的SQL语句中的参数，都有一个隐式的存储类。在下面描述的条件下，在查询执行阶段，数据库引擎可能会在数字存储类（INTEGER和REAL）和TEXT存储类之间转换。
{% blockquote %} 
1. Boolean数据类型
SQLite没有单独的Boolean存储类，相反，Booean值以整数0（false）和1（true）存储。
2. 日期和时间数据类型
SQLite没有为存储日期和/或时间设置专门的存储类，相反，内置的日期和时间函数能够把日期和时间作为TEXT，REAL或INTEGER值存储：
TEXT：作为ISO8601字符串（"YYYY-MM-DD HH:MM:SS.SSS"）。
 REAL：作为Julian天数，……
 INTEGER：作为Unix Time，即自1970-01-01 00:00:00 UTC以下的秒数。
{% endblockquote %}
 
### 类型相像（type affinity）
&emsp;&emsp;为了最大化SQLite和其他数据库引擎之间的兼容性，SQLite支持列的”类型相像“的概念。这里重要的思想是，类型是推荐的，不是必需的。任何列仍然能够存储任何类型的数据。只是某些列，能够选择优先使用某种存储类。这个对列的优先存储类称作它的”相像“。
 &emsp;&emsp;SQLite 3 数据库中的每个列都赋予下面类型相像中的一个：
 {% blockquote %} 
 TEXT
  NUMERIC
  INTEGER
  REAL
  NONE
  {% endblockquote %}
&emsp;&emsp;带有TEXT相像的列会使用存储类NULL、TEXT或BLOB来存储所有的数据。如果数据数据被插入到带有TEXT相像的列中，它会在插入前转换为文本格式。
&emsp;&emsp;带有NUMERIC相像的列可以使用所有5个存储类来包含值。当文本数据被插入到一个NUMERIC列，文本的存储类会被转换成INTEGER或REAL（为了优先），如果这个转换是无损的和可逆的话。如果TEXT到INTEGER或REAL的转换是不可能的，那么值会使用TEXT存储类存储。不会试图转换NULL或BLOB值。
 
 
#### 列相像的确定
&emsp;&emsp;列相像是由列声明的类型确定的，规则是按照下面的顺序：
     1. 如果声明的类型包含字符串”INT“那么它被赋予INTEGER相像。
     2. 如果列声明的类型包含任何字符串”CHAR“，”CLOB“，或”TEXT“，那么此列拥有TEXT相像。注意类型VARCHAR包含”CHAR“，因此也会赋予TEXT相像。
     3. 如果列声明的类型包含”BLOB“或没有指定类型，那么此列拥有NONE相像。
     4. 如果列声明的类型包含任何”REAL“，”FLOA“，或”DOUB“，那么此列拥有REAL相像。
     5. 其他情况，相像是NUMERIC。
 注意规则的顺序是重要的。声明类型为“CHARINT”的列同时匹配规则1和规则2，但第一个规则会优先采用，因此此列的相像是INTEGER。
 
#### 比较表达式
    Sqlite v3有一系列有用的比较操作符，包括 "=", "==", "<", "<=", ">", ">=", "!=", "<>", "IN", "NOT IN", "BETWEEN", "IS", 和 "IS NOT"


#### 排序
 &emsp;&emsp;比较操作的结果基于操作数的存储类型，根据下面的规则：
 
    存储类型为NULL的值被认为小于其他任何的值（包括另一个存储类型为NULL的值）;
    一个INTEGER或REAL值小于任何TEXT或BLOB值。当一个INTEGER或REAL值与另外一个INTEGER或REAL值比较的话，就执行数值比较;
    TEXT值小于BLOB值。当两个TEXT值比较的时候，就根据序列的比较来决定结果;
    当两个BLOB值比较的时候，使用memcmp来决定结果.
 
#### 比较操作数的近似（Affinity）
&emsp;&emsp;Sqlite可能在执行一个比较之前会在INTEGER，REAL或TEXT之间转换比较值。是否在比较操作之前发生转换基于操作数的近似（类型）。操作数近似（类型）由下面的规则决定：
         
      对一个列的简单引用的表达式与这个列有相同的affinity，注意如果X和Y.Z是列名，那么+X和+Y.Z均被认为是用于决定affinity的表达式
     一个”CAST(expr as type)”形式的表达式与用声明类型为”type”的列有相同的affinity
      其他的情况，一个表达式为NONE affinity

#### 在比较前的类型转换
&emsp;&emsp;只有在转换是无损、可逆转的时候“应用近似”才意味着将操作数转换到一个特定的存储类。近似在比较之前被应用到比较的操作数，遵循下面的规则（根据先后顺序）:
      
      如果一个操作数有INTEGER，REAL或NUMERIC近似，另一个操作数有TEXT或NONE近似，那么NUMERIC近似被应用到另一个操作数;
      如果一个操作数有TEXT近似，另一个有NONE近似，那么TEXT近似被应用到另一个操作数;
      其他的情况，不应用近似，两个操作数按本来的样子比较;
      表达式"a BETWEEN b AND c"表示两个单独的二值比较” a >= b AND a <= c”，即使在两个比较中不同的近似被应用到’a’。

#### 操作符
&emsp;&emsp;所有的数学操作符(+, -, *, /, %, <<, >>, &, |)，在被执行前，都会将两个操作数都转换为数值存储类型（INTEGER和REAL）。即使这个转换是有损和不可逆的，转换仍然会执行。一个数学操作符上的NULL操作数将产生NULL结果。一个数学操作符上的操作数，如果以任何方式看都不像数字，并且又不为空的话，将被转换为0或0.0。

&emsp;&emsp;有兴趣了解更多可以参考[这里](https://cloud.tencent.com/developer/section/1419985)


