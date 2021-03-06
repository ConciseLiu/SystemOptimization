# mysql中生成亿条数据用于测试
>计划mysql中生成千万条数据，传统方式耗时较长，且不稳定，查找资料后才用存储过程来生成数据

#### 1.生成思路
- 利用mysql内存表插入速度快的特点，先利用函数和存储过程在内存表中生成数据，然后再从内存表插入普通表中

#### 2.创建内存表及普通表

- 内存表
```mysql
CREATE TABLE `vote_record_memory` (
	`id` INT (11) NOT NULL AUTO_INCREMENT,
	`user_id` VARCHAR (20) NOT NULL,
	`vote_id` INT (11) NOT NULL,
	`group_id` INT (11) NOT NULL,
	`create_time` datetime NOT NULL,
	PRIMARY KEY (`id`),
	KEY `index_id` (`user_id`) USING HASH
) ENGINE = MEMORY AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8
```
- 普通表
```mysql

CREATE TABLE `vote_record` (
	`id` INT (11) NOT NULL AUTO_INCREMENT,
	`user_id` VARCHAR (20) NOT NULL,
	`vote_id` INT (11) NOT NULL,
	`group_id` INT (11) NOT NULL,
	`create_time` datetime NOT NULL,
	PRIMARY KEY (`id`),
	KEY `index_user_id` (`user_id`) USING HASH
) ENGINE = INNODB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8

```
#### 3.创建函数及存储过程

```mysql
CREATE FUNCTION `rand_string`(n INT) RETURNS varchar(255) CHARSET latin1
BEGIN 
DECLARE chars_str varchar(100) DEFAULT 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'; 
DECLARE return_str varchar(255) DEFAULT '' ;
DECLARE i INT DEFAULT 0; 
WHILE i < n DO 
SET return_str = concat(return_str,substring(chars_str , FLOOR(1 + RAND()*62 ),1)); 
SET i = i +1; 
END WHILE; 
RETURN return_str; 
END
```

```mysql

CREATE  PROCEDURE `add_vote_memory`(IN n int)
BEGIN  
  DECLARE i INT DEFAULT 1;
    WHILE (i <= n ) DO
      INSERT into vote_record_memory  (user_id,vote_id,group_id,create_time ) VALUEs (rand_string(20),FLOOR(RAND() * 1000),FLOOR(RAND() * 100) ,now() );
			set i=i+1;
    END WHILE;
END
```

#### 4、调用存储过程

```mysql
CALL add_vote_memory(1000000)
```

> 根据电脑性能不能所花时间不一样，大概时间在小时级别，如果报错内存满了，只在修改max_heap_table_size 个参数即可，win7修改位置如下，linux，修改my.cnf文件，修改后要重启mysql，重启后内存表数据会丢失

#### 5、调用存储过程

```mysql
INSERT into vote_record SELECT * from  vote_record_memory
```

#### 6. 优化mysql参数，加快插入速度

```mysql
# 配置文件(my.ini)中新增该选项
innodb_flush_log_at_trx_commit=0

```
innodb_flush_log_at_trx_commit的相关解释
>If the value of innodb_flush_log_at_trx_commit is 0, the log buffer is written out to the log file once per second and the flush to disk operation is performed on the log file, but nothing is done at a transaction commit. When the value is 1 (the default), the log buffer is written out to the log file at each transaction commit and the flush to disk operation is performed on the log file. When the value is 2, the log buffer is written out to the file at each commit, but the flush to disk operation is not performed on it. However, the flushing on the log file takes place once per second also when the value is 2. Note that the once-per-second flushing is not 100% guaranteed to happen every second, due to process scheduling issues. 

>参考网站：
https://blog.csdn.net/whzhaochao/article/details/49126037
https://www.cnblogs.com/suifengbingzhu/p/4182985.html
https://www.zhihu.com/question/19719997