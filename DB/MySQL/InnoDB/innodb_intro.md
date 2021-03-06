---
# InnoDB存储引擎结构介绍
---

## Ⅰ、InnoDB发展史
|时间|事件|备注|
|:-:|:-:|:-:|
|1995|由Heikki Tuuri创建Innobase Oy公司，开发InnoDB存储引擎|Innobase开始做的是数据库，希望卖掉该公司|
|1996|MySQL 1.0 发布||
|2000|MySQL3.23版本发布||
|2001|InnoDB存储引擎集成到MySQL数据库|以插件方式集成|
|2006|Innobase被Oracle公司收购（InnoDB作为开源产品，性能和功能很强大）|InnoDB在被收购后的，MySQL中的InnoDB版本没有改变|
|2010|MySQL5.5版本InnoDB存储引擎称为默认存储引擎|MySQL被Sun收购，Sun被Oracle收购，使得MySQL和InnoDB重新在一起配合开发|
|至今|其他存储引擎已经不再得到Oracle官方的后续开发||

## Ⅱ、InnoDB重要特性一览
- Fully ACID（InnoDB默认的Repeat Read隔离级别就支持）
- Row-level Locking（支持行锁）
- Multi-version concurrency control（MVCC）（支持多版本并发控制）
- Foreign key support（支持外键）
- Automatic deadlock detection（死锁自动检测）
- High performance、High scalability、High availability（高性能，高扩展，高可用）

## Ⅲ、物理存储结构
### 主要组成部分

**表空间文件：**

- 独立表空间
- 共享表空间
- undo表空间(MySQL5.6开始)

**重做日志文件：**

- 物理逻辑日志
- 没有Oracle的归档重做日志

### 细说表空间文件

**表空间的概念：**

- 表空间是一个逻辑存储的概念
- 表空间可以由多个文件组成
- 支持裸设备（可以直接使用O_DIRECT 方式绕过缓存，直接写入磁盘）

**表空间的分类：**

①共享表空间(最早只有这个)
- 存储元数据信息
- 存储Change Buffer信息
- 存储Undo信息
- MySQL4.0之前所有数据都是存储在共享表空间中

②独立表空间
- MySQL4.0开始，支持每张表对应一个独立表空间(ibd文件)
- innodb-file-per-table=1（默认为1，这个参数关掉创建表，会发现对应的库下面没有该表的idb文件）
- 分区表可以对应多个ibd文件

③undo表空间
- MySQL5.6版本支持独立的Undo表空间,默认是0，即undo记录在共享表空间中
- innodb_undo_tablespaces(该值8.0开始将会被剔除，不可修改，默认写死，为2)

④临时表空间
- MySQL5.7增加了临时表空间（ibtmp1）
- innodb_temp_data_file_path

### 看下数据目录
```
[root@VM_0_5_centos data3306]# ll ib*
-rw-r----- 1 mysql mysql    16285 Feb  4 18:15 ib_buffer_pool
-rw-r----- 1 mysql mysql 79691776 Feb 24 10:53 ibdata1          #共享表空间
-rw-r----- 1 mysql mysql 50331648 Feb 24 10:53 ib_logfile0      #重做日志
-rw-r----- 1 mysql mysql 50331648 Feb  4 15:06 ib_logfile1    
-rw-r----- 1 mysql mysql 12582912 Feb 24 10:14 ibtmp1           #临时表空间

两个ib_logfile是循环交替写入的，SSD下尽可能设大，单个文件4G/8G，设置太小可能会导致脏页刷新时hang住

MySQL中现在只有这两个文件不会归档，最老的3.23版本支持归档，因为MySQL有二进制日志所以把这个功能阉割了

[root@VM_0_5_centos data3306]# cd test
[root@VM_0_5_centos test]# ll
total 228
-rw-r----- 1 mysql mysql       65 Feb  4 13:21 db.opt           #记录默认字符集和字符集排序规则
-rw-r----- 1 mysql mysql  8554 Feb  1 15:45 abc.frm             #表结构文件
-rw-r----- 1 mysql mysql 98304 Feb 24 10:53 abc.ibd             #独立表空间
```

就性能而言，独立和共享速度是一样的，基本上区别

虽然ibadta1是一个文件，但是在底层申请也是1兆1兆地申请的空间，是连续的(两种都是以区的方式来管理)，在磁盘上的表现是一样的，1M的数据基本上可以认为是连续的

**tips:**

sysbench去测8个文件开16个线程和测1个文件开128个线程，测出来iops是一样的，因为它的分配和管理机制都是一样的

那为什么MySQL4.0版本开始引入了独立表空间呢？

主要是为了管理方便，表现为：

- 看上去清晰
- 删除文件非常简单，drop完空间可释放，对于ibdata1来说，一个文件，只能增不能减，删除一张表只是把这张表对应的空间标为可用，但是空间并不能回收
删除.ibd文件是不行的，因为对应的innodb元数据表里面的数据没有删

.ibd和.ibdata1文件坏了修复的话收费是按行数来算的很贵的哦，相对而言ibdata1文件修复难度更大

切记：不要在数据目录下删除任何文件

### 小常识
单个 ibd文件 直接拷贝到新的数据库中无法直接恢复：
- 原因一：元数据信息还是在ibdata1中
- 原因二：部分索引文件存在于Change Buffer中，目前还是存放于ibdata1文件中

查看表空间的元数据信息
```
select * from information_schema.innodb_sys_tablespaces;
```

## Ⅳ、逻辑存储结构
|表空间|
|:-:|
|内部有多个段对象（Segment）组成|
|每个段（Segment）由区（Extent）组成|
|每个区（Extent）由页（Page）组成|
|每个页里面保存数据（或者叫记录Row）|

**段：**

段对用户来说是透明的，也是一个逻辑的概念，用来组织管理区

在MySQL系统表中是看不到段这个元数据的，但是逻辑上的确存在(重点理解区和页)

**区：**

区包含页

区是最小的空间申请单位，表空间文件要扩展是以区的单位来扩展

区的大小固定为1M，这1M在物理上是连续的，但1M和1M之间不保证连续

page_size=16K 就是1M * 1024 / 16 = 64个页

16k 64个页
8k  128个页
4k  256个页

通常一次申请4个区大小，1兆1兆地申请，特殊情况会申请5个区，很少发生这种情况


**页 :**

等价于ORACLE中的块 ,最小的I/O操作单位

**tips:**

data的最小单位不是页，而是页中的记录(row)

普通用户表中MySQL默认的每个页为16K
- 从MySQL 5.6开始使用 innodb_page_size可以控制页大小
- 一旦数据库通过innodb_page_size创建完成，则后续无法更改
- innodb_page_size 是针对普通表的,压缩表不受其限制

### 来来来，吹两手
从5.6开始可以调整大小，但只能设置4k和8k，从5.7开始还可以设置32k，64k，如果设置为32k或者64k那区的大小就变为2M和4M

什么淘宝标准MysQL配置参数把这个大小设置为4k，完全扯淡，就算在ssd下4k可能也不会比16k好，另外设置为4k，io操作会变多，不要迷信，现在的表偏向于宽表(很多列组成，单行记录比较大）,一行超过1k甚至4k以上，这时候这个页大小设为4，性能会很差。

设置小了，B+ tree高度变高，io变多，性能变差;如果是宽列，4k的页会导致行外存，效率变差

目前为止，大家都用的SSD，列比较宽，内存也比较大，就保留16k或者尝试一下32k(一些单列特别大的场景，大页性能有优势)

**tips:**

MySQL页大小和ORACLE页大小不同的是，MySQL页大小是全局的，一旦初始化好，就不可再修改，不像ORACLE可以设置每张表的页大小

## Ⅴ、MySQL中如何定位到一个页

**SpaceID**

- 每个表空间都对应一个SpaceID，而表空间又对应一个ibd文件，那么一个ibd文件也对应一个SpaceID
- ibdata1对应的SpaceID为0，每创建一个表空间（ibd文件），SpaceID自增长（全局）

**PageNumber**

- 在一个表空间中，第几个16K的页（假设 innodb_page_size = 16K）即为PageNumber

```
               +--------------------+--------------+-------------+--------------+
      SpaceID  |         0          |      1       |             |      N       | # 全局递增              
	       +----------------------------------------------------------------+              
	       |                    |              ||           ||              |
      ibd File |      ibdata1       |  table1.ibd  ||  ......   ||  tableX.ibd  +------+              
	       |                    |              ||           ||              |      |              
	       +--------+-----------+-------+-----------------------------------+      |                       
		        |                   |                                          |                       
			|                   |                                          |                       
			|                   +--------------------------------------+   |                       
	                |                                                          |   |                       
			|                                                          |   |               
	       +--------v---+------------+--------------------------------------+  |   |
	       |            |            |             ||          ||           |  |   |              
               |     16K    |     16K    |     16K     ||  ......  ||    16K    |  |   |              
	       |            |            |             ||          ||           |  |   |              
	       +----------------------------------------------------------------+  |   |
Page Number    |      0     |      1     |      2      |            |     M     |  |   |                
	       +------------+------------+-------------+------------+-----------+  |   |                                                                                                                                    |   |                                                                                                                                    |   |                       
	                +----------------------------------------------------------+   |                       
			|                                                              |                       
			|                                                              |                       
		        |                                                              |                     
			|                                                              |              
	       +--------v---+------------+--------------------------------------+      |              
	       |            |            |             ||          ||           |      |              
               |     16K    |     16K    |     16K     ||  ......  ||    16K    |      |              
	       |            |            |             ||          ||           |      |              
	       +----------------------------------------------------------------+      | 
Page Number    |      0     |      1     |      2      |            |     M     |      |		 
	       +------------+------------+-------------+------------+-----------+      |                                                                                                                                        |   
                                                                                       |
		        +--------------------------------------------------------------+                       
		        |                       
                        |                     
			|              
	       +--------v---+------------+--------------------------------------+                          
	       |            |            |             ||          ||           |              
	       |     16K    |     16K    |     16K     ||  ......  ||    16K    |              
	       |            |            |             ||          ||           |              
	       +----------------------------------------------------------------+ 
Page Number    |      0     |      1     |      2      |            |     M     |              
	       +------------+------------+-------------+------------+-----------+              
					   # 每个表空间中，都是从0开始递增，且仅仅是表空间内唯一


```

- 每次读取Page时，都是通过SpaceID和PageNumber进行读取
- 可以简单理解为从表空间的开头读多少个PageNumber * PageSize的字节（偏移）
- 想成数组，数组的名字就是SpaceID，数组的下标就是PageNumber
- 在一个SpaceID（ibd文件）中，PageNumber是唯一且自增的
- 删除表的时候，SpaceID不会回收 ，SpaceID是全局自增长的

**tips：**
这里的区（extent）的概念已经弱化

在这个例子中，第一个区的PageNumber是（0到63）且这64个页在物理上是连续的；第二个区的PageNumber是（64到127） 且这64个页在物理上也是连的

但是（0到63）和（64到127）之间在物理上则不一定是连续的，因为区和区之间在物理上不一定是连续的

### 随便看看
```
(root@localhost) [information_schema]> select space, name from information_schema.innodb_sys_tablespaces order by space limit 5;
+-------+---------------------+
| space | name                |
+-------+---------------------+
|     2 | mysql/plugin        |
|     3 | mysql/servers       |
|     4 | mysql/help_topic    |
|     5 | mysql/help_category |
|     6 | mysql/help_relation |
+-------+---------------------+
5 rows in set (0.00 sec)

(root@localhost) [information_schema]> select name, space, table_id from information_schema.innodb_sys_tables where space=0;
+------------------+-------+----------+
| name             | space | table_id |
+------------------+-------+----------+
| SYS_DATAFILES    |     0 |       14 |
| SYS_FOREIGN      |     0 |       11 |
| SYS_FOREIGN_COLS |     0 |       12 |
| SYS_TABLESPACES  |     0 |       13 |
| SYS_VIRTUAL      |     0 |       15 |
+------------------+-------+----------+
5 rows in set (0.00 sec)

(root@localhost) [information_schema]> select name, space, table_id from information_schema.innodb_sys_tables where space<>0 order by space limit 5;
+---------------------+-------+----------+
| name                | space | table_id |
+---------------------+-------+----------+
| mysql/plugin        |     2 |       16 |
| mysql/servers       |     3 |       17 |
| mysql/help_topic    |     4 |       35 |
| mysql/help_category |     5 |       36 |
| mysql/help_relation |     6 |       38 |
+---------------------+-------+----------+
5 rows in set (0.00 sec)
```

- 独立表空间的table_id和SpaceID一一对应 
- 共享表空间是多个table_id对应一个SpaceID
- SpaceID为0的是ibdata1，1这个位置没有，空的
