# Mysql Innodb 死锁
最近William发现业务上有很多死锁，运行`SHOW ENGINE INNODB STATUS`结果如下(省略了若干log)：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2017-08-28 17:10:14 7f7cc14b2700
*** (1) TRANSACTION:
TRANSACTION 441390816, ACTIVE 5.089 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 9 lock struct(s), heap size 2936, 158 row lock(s)
LOCK BLOCKING MySQL thread id: 352667151 block 403468435
MySQL thread id 403468435, OS thread handle 0x7f7ca8e38700, query id 333255350 10.25.1.254 xmonster update
INSERT INTO t_user_post_score (f_user_id, f_post_id, f_bit_relation, f_create_time) VALUES (198425,175660,8,'2017-08-28 17:10:09'),(198425,175798,8,'2017-08-28 17:10:09'),(198425,173936,8,'2017-08-28 17:10:09'),(198425,171842,8,'2017-08-28 17:10:09'),(198425,184665,8,'2017-08-28 17:10:09'),(198425,180258,8,'2017-08-28 17:10:09'),(198425,177120,8,'2017-08-28 17:10:09'),(198425,175850,8,'2017-08-28 17:10:09'),(198425,173283,8,'2017-08-28 17:10:09'),(198425,172530,8,'2017-08-28 17:10:09'),(198425,172333,8,'2017-08-28 17:10:09'),(198425,193575,8,'2017-08-28 17:10:09'),(198425,177090,8,'2017-08-28 17:10:09'),(198425,174874,8,'2017-08-28 17:10:09'),(198425,197104,8,'2017-08-28 17:10:09'),(198425,195203,8,'2017-08-28 17:10:09'),(198425,175508,8,'2017-08-28 17:10:09'),(198425,174616,8,'2017-08-28 17:10:09'),(198425,174577,8,'2017-08-28 17:10:09'),(198425,173888,8,'2017-08-28 17:10:09'),(198425,192733,8,'2017-08
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 2547 page no 35255 n bits 384 index `PRIMARY` of table `letsgo`.`t_user_post_score` trx id 441390816 lock_mode X locks rec but not gap waiting
Record lock, heap no 159 PHYSICAL RECORD: n_fields 10; compact format; info bits 0
 0: len 4; hex 863fc6c7; asc  ?  ;;
 1: len 6; hex 00001a41a787; asc    A  ;;
 2: len 7; hex 44000080370bcb; asc D   7  ;;
 3: len 4; hex 80030719; asc     ;;
 4: len 4; hex 80030d13; asc     ;;
 5: len 4; hex 7ffffc18; asc     ;;
 6: len 4; hex 59a370eb; asc Y p ;;
 7: len 4; hex 59a370f5; asc Y p ;;
 8: len 0; hex ; asc ;;
 9: len 4; hex 8000000c; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 441388379, ACTIVE 56.968 sec fetching rows
mysql tables in use 2, locked 2
28778 lock struct(s), heap size 3569192, 1549 row lock(s)
MySQL thread id 352667151, OS thread handle 0x7f7cc14b2700, query id 333142426 10.25.1.254 xmonster Sending data
UPDATE t_user_post_score, t_post_score SET t_user_post_score.f_score = 1000 * t_post_score.f_score WHERE t_user_post_score.f_post_id = t_post_score.f_post_id AND t_user_post_score.f_post_id in (200257,199955,199851,200104)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 2547 page no 35255 n bits 384 index `PRIMARY` of table `letsgo`.`t_user_post_score` trx id 441388379 lock_mode X locks rec but not gap
Record lock, heap no 47 PHYSICAL RECORD: n_fields 10; compact format; info bits 0
 0: len 4; hex 863fc657; asc  ? W;;
 1: len 6; hex 00001a41a5e6; asc    A  ;;
 2: len 7; hex 450000619e1172; asc E  a  r;;
 3: len 4; hex 800472d7; asc   r ;;
 4: len 4; hex 80030d13; asc     ;;
 5: len 4; hex 7ffffc18; asc     ;;
 6: len 4; hex 59a370e2; asc Y p ;;
 7: len 4; hex 59a370e2; asc Y p ;;
 8: len 0; hex ; asc ;;
 9: len 4; hex 80000008; asc     ;;
```

可以从上面看到一个update和insert语句造成了DEADLOCK，可以search一下相关的文章其实不少，但大多是关于外键造成的。[这边文章](https://www.xaprb.com/blog/2006/08/03/a-little-known-way-to-cause-a-database-deadlock/)中举的例子和我们遇到的情况十分类似。

例子中表为:
```
create table ad_data (
   ad int not null,
   day date not null,
   impressions int not null,
   clicks int not null,
   cost_cents int not null,
   avg_pos decimal(3, 1) not null,
   primary key (day, ad)
) engine = InnoDB;
```

语句1：
```
create temporary table cost as
   select day, client, sum(clicks), sum(cost)
      from ad_data
      where day = '2006-08-01'
      group by day, client;
```

语句2:
```
insert into ad_data(day, ad_id, client, clicks, cost) VALUES
  ('2006-08-01', 7, 1, 70, 700),
  ('2006-08-01', 5, 1, 50, 500);
```

分析如上例子：2句sql都会得到主键的锁，请InnoDB的主键为[clustered index](https://www.xaprb.com/blog/2006/07/04/how-to-exploit-mysql-index-optimizations/)。语句1会依次得到符合where的所有行的锁，请注意这里是依次而不是原子的获得所有。而语句2则会插入语句1锁范围内的记录，如果语句1仅仅是select,则会出现[Phantom Read](https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html)。而这里是增加了写锁，所以T1的锁是禁止T2进入的，实际的情况可能会如下图所示:

![deadlock](https://d33wubrfki0l68.cloudfront.net/eb0c70428664b78d61c3f23c4e4a0a24be27501b/f7825/media/2006/08/deadlock.png)

这里主要造成问题的是因为2个语句得到锁的方向是不同的，是有造成死锁的风险的。这里我们可以总结，**如果我们总是按主键的方向去获取锁**，是可以大大降低死锁的概率的。另外 **如果语句2是每句insert单独获取锁** 也可以避免死锁.
