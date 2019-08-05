---
layout:     post
title:      InnoDB和MyISAM的区别
subtitle:   
date:       2019-4-12
author:     BY xukexiang
header-img: img/charlotte/b2f68e7ebd6314a8358661a765ca9095527eeee1.jpg
catalog: true
tags:
    - Typora
---


#### 1、事务支持

MyISAM不支持事务，而InnoDB支持事务，InnoDB的AUTOCOMMIT默认是打开的，
即每条SQL语句都会被封装成一个事务自动提交


#### 2、存储结构
MyISAM：每个MyISAM在磁盘上存储成三个文件。第一个文件以表的名字开始，扩展名
指出文件类型，.frm文件存储表定义。数据文件的扩展名为MYD。索引文件的扩展名为MYI。

InnoDB：所有表都保存在同一个数据文件中。

#### 3、存储空间
MyISAM：可以被压缩，存储空间小。支持三种不同的存储格式：静态表(默认，但是注意数据末尾不能有空格)、动态表，压缩表。

InnoDB：需要更多的内存和存储，它会在主内存中建立专用的缓冲池用于高速缓冲数据和索引。

#### 4、外键
MyISAM：不支持

InnoDB：支持

#### 5、表锁差异
MyISAM：只支持表级锁，用户在操作myisam表时，select，update，delete，insert语句都会给表自动加锁，如果加锁以后的表满足insert并发的情况下，可以在表的尾部插入新的数据。

InnoDB：支持事务和行级锁，是innodb的最大特色。行锁大幅度提高了多用户并发操作的新能。
但是InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表的。

MyISAM锁的粒度是表级，而InnoDB支持行级锁定。简单来说就是, 
InnoDB支持数据行锁定，而MyISAM不支持行锁定，只支持锁定整个表。
即MyISAM同一个表上的读锁和写锁是互斥的，MyISAM并发读写时如果等待队列中既有读请求又有写请求，
默认写请求的优先级高，即使读请求先到，所以MyISAM不适合于有大量查询和修改并存的情况，
那样查询进程会长时间阻塞。因为MyISAM是锁表，所以某项读操作比较耗时会使其他写进程饿死。


#### 6、查询效率

没有where的count(*)使用MyISAM要比InnoDB快得多。因为MyISAM内置了一个计数器，count(*)时它直接从计数器中读，
而InnoDB必须扫描全表。所以在InnoDB上执行count(*)时一般要伴随where，且where中要包含主键以外的索引列。
为什么这里特别强调“主键以外”？因为InnoDB中primary index是和raw data存放在一起的，
而secondary index则是单独存放，然后有个指针指向primary key。
所以只是count(*)的话使用secondary index扫描更快，
而primary key则主要在扫描索引同时要返回raw data时的作用较大。
MyISAM相对简单，所以在效率上要优于InnoDB，小型应用可以考虑使用MyISAM。