---
title: MySQL Note
date: 2019-02-20
categories:
- Notes
- SQL
tags:
- SQL
---

## This are some notes for mysql. 

<!--more-->



## optimize table
> 資料庫使用久了，會開始產生不連續碎片(fragmented), 透過 optimize 重組(Vacuum)

```sql=
OPTIMIZE TABLE table_name;
```
```
+------------------+----------+----------+-------------------------------------------------------------------+
| Table            | Op       | Msg_type | Msg_text                                                          |
+------------------+----------+----------+-------------------------------------------------------------------+
| Prognostics.data | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| Prognostics.data | optimize | status   | OK                                                                |
+------------------+----------+----------+-------------------------------------------------------------------+
```
正常會是: status ok (MyISAM)
但 InnoDB 會是以上的狀況, 原因如下:

> For InnoDB tables, *OPTIMIZE TABLE is mapped to ALTER TABLE*, which rebuilds the table to update index statistics and free unused space in the clustered index.
也就是, innoDB 中就會轉換為以下語法:
```
ALTER TABLE table.name ENGINE='InnoDB';
```
這語法的作用會是: 建立一個新的 Table, 由舊的 Table 將資料拷貝進來, 然後再把舊的 Table 砍掉, 但是, 作者建議先備份後再來執行此動作比較好.
結論: 用 optimize table 就好惹

ref:

 - [MySQL 對 MyISAM、InnoDB 使用 Optimize Table](https://blog.longwin.com.tw/2012/03/mysql-myisam-innodb-optimize-2012/)

 - [实例说明optimize table在优化mysql时很重要](http://blog.51yip.com/mysql/1222.html)


## 資料庫調校

```sql
innodb_log_file_size = 512M

innodb_log_buffer_size = 128M

innodb_flush_log_at_trx_commit = 0
-- speed: 0>2>1
innodb_buffer_pool_size = 2G

innodb_additional_mem_pool_size = 32M

innodb_data_file_path = ibdata1:1G:autoextend

innodb_flush_method=O_DIRECT
-- only for linux
query_cache_size=32M

thread_cache_size = 16
```

## 清理碎片

1. InnoDB 用下面這個指令就可以重新把碎片清掉
```sql
ALTER TABLE finding engine=InnoDB;

-- -- for MyISAM
optimize table finding; 
```
2. 確認表格中碎片的大小
```sql
-- data_free 就是碎片大小
SHOW TABLE STATUS LIKE 'finding';


-- 清查所有表的大小和碎片大小
SELECT table_name, round(((data_length + index_length) / 1024 / 1024), 2) as table_size, round(((data_free) / 1024 / 1024), 2) as fragmentation_size
                      FROM information_schema.TABLES WHERE table_schema="Prognostics" 
                      ORDER BY (data_length + index_length) DESC

```
![](https://i.imgur.com/SEhEaeC.png)

-- 資料庫中每張表的表格大小以及對應的碎片大小
![](https://i.imgur.com/H00GwfG.png)


## 資料庫大動作變動程序 (盡可能不要影響到原表格)

```sql=
SET GLOBAL storage_engine=InnoDB;

-- 複製備份資料
create table finding_tmp as select * from finding ;

-- 重新設定 pk
ALTER TABLE finding_tmp ADD PRIMARY KEY(id);

-- 主要變更步驟: 修正 schema 以及 column type
ALTER TABLE finding_tmp MODIFY COLUMN id BIGINT AUTO_INCREMENT;

-- 確認如果表格 type 已經是 InnoDB 就可以少做這一步
ALTER TABLE finding_tmp engine=InnoDB;

-- InnoDB 才有支援 fk 最後補上
ALTER TABLE finding_tmp ADD CONSTRAINT fk_miningId FOREIGN KEY (miningId) REFERENCES mining(id) ON DELETE CASCADE;
ALTER TABLE finding_tmp ADD CONSTRAINT fk_responseId FOREIGN KEY (responseId) REFERENCES data(id) ON DELETE CASCADE;
ALTER TABLE finding_tmp ADD CONSTRAINT fk_predictorId FOREIGN KEY (predictorId) REFERENCES data(id) ON DELETE CASCADE;
ALTER TABLE finding_tmp ADD CONSTRAINT fk_timetrendId FOREIGN KEY (timetrendId) REFERENCES data(id) ON DELETE CASCADE;
ALTER TABLE finding_tmp ADD CONSTRAINT fk_predictorEquipmentId FOREIGN KEY (predictorEquipmentId) REFERENCES data(id) ON DELETE CASCADE;

-- 新的表格確定無誤之後, 直接交換表格名稱即可
RENAME TABLE finding TO tmp_table,
finding_tmp TO finding,
tmp_table TO finding_backup;
```

## 確認表格的 foreign key

```sql=
SELECT 
  TABLE_NAME,COLUMN_NAME,CONSTRAINT_NAME, REFERENCED_TABLE_NAME,REFERENCED_COLUMN_NAME
FROM
  INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE
  REFERENCED_TABLE_SCHEMA = 'Prognostics' AND
  TABLE_NAME = 'finding';
```

ref:

 - [MySQL數據丟失討論](https://read01.com/zh-tw/jjjzjE.html#.WiZ_h0j0BhE)

 - [innodb的innodb_buffer_pool_size和MyISAM的key_buffer_size](http://www.cnblogs.com/zsmynl/p/3602922.html)

 - [60 million entries, select entries from a certain month. How to optimize database?](https://stackoverflow.com/questions/5451190/60-million-entries-select-entries-from-a-certain-month-how-to-optimize-databas/5451389#5451389)

 - [Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)

 - [How to exploit MySQL index optimizations](https://www.xaprb.com/blog/2006/07/04/how-to-exploit-mysql-index-optimizations/)

 - [What to tune in MySQL Server after installation](https://www.percona.com/blog/2006/09/29/what-to-tune-in-mysql-server-after-installation/)

 - [Choosing innodb_buffer_pool_size](https://www.percona.com/blog/2007/11/03/choosing-innodb_buffer_pool_size/)
