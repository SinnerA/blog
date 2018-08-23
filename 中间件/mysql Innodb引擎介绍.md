---
title: mysql Innodb引擎介绍
date: 2018-08-13
tags: 
    - mysql
---

[TOC]

## MyISAM 和 InnoDB 的区别

**区别：**

1. InnoDB支持事务，MyISAM不支持，对于**InnoDB每一条SQL语言都默认封装成事务，自动提交**，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务；  
2. InnoDB支持外键，而MyISAM不支持。对一个包含外键的InnoDB表转为MYISAM会失败；  
3. InnoDB是聚集索引，数据文件是和索引绑在一起的，必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而MyISAM是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。 
4. InnoDB不保存表的具体行数，执行select count(*) from table时需要全表扫描。而MyISAM用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快；  
5. Innodb不支持全文索引，而MyISAM支持全文索引，查询效率上MyISAM要高（mysql5.7的innodb已支持全文索引）
6. InnoDB支持行级锁


**如何选择：**

1. 是否要支持事务，如果要请选择innodb，如果不需要可以考虑MyISAM；
2. 如果表中绝大多数都只是读查询，可以考虑MyISAM，如果既有读写也挺频繁，请使用InnoDB。
3. 系统奔溃后，MyISAM恢复起来更困难，能否接受；
4. MySQL5.5版本开始Innodb已经成为Mysql的默认引擎(之前是MyISAM)，说明其优势是有目共睹的，如果你不知道用什么，那就用InnoDB，至少不会差。

简单说

读操作多用**MyISAM**
**写操作多用**InnoDB

InnoDB每一条SQL语言都默认封装成事务，因此对于select语句来说，MyISAM要优于InnoDB

## 参考

[Mysql 中 MyISAM 和 InnoDB 的区别有哪些？](https://www.zhihu.com/question/20596402)





