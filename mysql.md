

```sql

-- json_table
-------------------
> select * from json_table('[1,2,3]', '$[*]' columns (
>     id bigint path '$'
> )) as t;
----
id
---
1
2
3
---

select * from json_table('{"lineId":1,"points":[{"name":"p#0","radius":39},{"name":"p#1","radius":95}]}', 
  '$.points[*]' COLUMNS (
               name VARCHAR(255) PATH '$.name',
               radius INT PATH '$.radius'
)) as x;
---------------
name | radius
---------------
p#0	39
p#1	95
------------------

mysql> SELECT * 
    -> FROM 
    -> JSON_TABLE( 
    -> '[ {"a": 1, "b": [11,111]}, {"a": 2, "b": [22,222]}, {"a":3}]', 
    -> '$[*]' COLUMNS( 
    -> a INT PATH '$.a', 
    -> NESTED PATH '$.b[*]' COLUMNS (b INT PATH '$') 
    -> ) 
    -> ) AS jt 
    -> WHERE b IS NOT NULL; 
+------+------+ 
| a    | b    | 
+------+------+ 
| 1    |   11 | 
| 1    |  111 | 
| 2    |   22 | 
| 2    |  222 | 
+------+------+

-- json_arrayagg  
-- JSON_OBJECTAGG(key, value) [over_clause]
-- 和group_concat差不多, json_arrayagg不能使用distinct,可以使用json_objectagg

> select json_arrayagg(name),group_concat(distinct name) from tablex group by id

-- json_objectagg
mysql> SELECT o_id, attribute, value FROM t3;
+------+-----------+-------+
| o_id | attribute | value |
+------+-----------+-------+
|    2 | color     | red   |
|    2 | fabric    | silk  |
|    3 | color     | green |
|    3 | shape     | square|
+------+-----------+-------+
4 rows in set (0.00 sec)

mysql> SELECT o_id, JSON_OBJECTAGG(attribute, value)
    -> FROM t3 GROUP BY o_id;
+------+---------------------------------------+
| o_id | JSON_OBJECTAGG(attribute, value)      |
+------+---------------------------------------+
|    2 | {"color": "red", "fabric": "silk"}    |
|    3 | {"color": "green", "shape": "square"} |
+------+---------------------------------------+
2 rows in set (0.00 sec)
```

### backup/restore

```bash
# 查看glibc的版本
ldd --version


# create readonly user
CREATE USER `ro`@`%` IDENTIFIED BY 'readonly@pwd';

GRANT Select, Show Databases, Show View ON *.* TO `ro`@`%`
GRANT Select, Show Databases, Show View,BACKUP_ADMIN,PROCESS,RELOAD, REPLICATION CLIENT ON *.* TO 'ro'@'%';
FLUSH PRIVILEGES;

# backup by xtrabackup
# 全量备份
./xtrabackup  \
--user=ro \
--password='readonly@pwd' \
--host=127.0.0.1 \
--port=3306 \
--backup \
--target-dir=/data/backup/fullmysql \
--no-lock  \
--datadir=/data/mysql/data

# increment
# 增量备份
./xtrabackup  \
--user=ro \
--password='readonly@pwd' \
--host=127.0.0.1 \
--port=3306 \
--backup \
--target-dir=/data/backup/incrmysql1 \
--no-lock  \
--datadir=/data/mysql/data \
--incremental-basedir=/data/backup/fullmysql

# restore
# 恢复,恢复全量备份,如果没有--apply-log-only则不会进行增量恢复
./xtrabackup --prepare --target-dir=/data/backup/fullmysql

# increment
# 恢复,使用增量方式恢复,可以使用增量的备份的数据一直恢复
./xtrabackup --prepare --apply-log-only \
--target-dir=/data/backup/fullmysql \

# 基于全量的备份进行增量恢复
./xtrabackup --prepare --apply-log-only \
--target-dir=/data/backup/fullmysql \
--incremental-dir=/data/backup/incrmysql1

# copy to other server
# /home/data/mysqlbackup/dev/20260123
rsync -avz 20260123 root@192.168.1.3:/home/local/data/mysql/

# test by docker
docker run \
--name mysql \
-v /home/local/data/mysql/20260123:/var/lib/mysql \
-v /home/local/data/mysql/conf:/etc/mysql/conf.d \
-p 3306:3306 \
-d mysql:8.0.34 \
--character-set-server=utf8mb4 \
--collation-server=utf8mb4_unicode_ci \
--lower_case_table_names=1


# backup by mysqldump
mysqldump \
-h xxx \
-P 3306 \
-u ro \
-p 'readonly@pwd' \
--single-transaction \
--skip-lock-tables \
--all-databases \
> /home/local/data/mysql/all.20260123.sql


# all
# backup,全量备份
./xtrabackup  --user=ro --password='xx@2025.RD' --host=192.168.1.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/full_20260123 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4

rsync -avz full_20260123 root@192.168.1.5:/home/local/data/mysql/dev/

# 第一个增量备份,基于全量备份
./xtrabackup  --user=ro --password='xx@2025.RD' --host=192.168.1.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/ince_1 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4 --incremental-basedir=/home/data/mysqlbackup/dev/full_20260123

# 第二次增量备份,基于前一个增量备份
./xtrabackup  --user=ro --password='x@2025.RD' --host=192.168.6.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/ince_2 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4 --incremental-basedir=/home/data/mysqlbackup/dev/ince_1

rsync -avz ince_1 192.168.1.5:/home/local/data/mysql/dev/
rsync -avz ince_2 192.168.1.5:/home/local/data/mysql/dev/

# restore, 恢复前需把全量的备份变成prepare,注意这个是不可逆的,且只能执行一次
./xtrabackup --prepare --apply-log-only --target-dir=/home/local/data/mysql/dev/full_20260123

# 恢复增量备份到全量备份
./xtrabackup --prepare --apply-log-only --target-dir=/home/local/data/mysql/dev/full_20260123 --incremental-dir=/home/local/data/mysql/dev/ince_1

# 恢复第二个增量备份到全量备份
./xtrabackup --prepare --apply-log-only --target-dir=/home/local/data/mysql/dev/full_20260123 --incremental-dir=/home/local/data/mysql/dev/ince_2

# 恢复最后一个增量备份到全量备份,没有加--apply-log-only则全量备份会被锁定,后续不可再通过增量备份恢复,只能执行一次
./xtrabackup --prepare --target-dir=/home/local/data/mysql/dev/full_20260123 --incremental-dir=/home/local/data/mysql/dev/ince_2

# 恢复全量备份, 后续不可在通过增量备份恢复,只能执行一次
./xtrabackup --prepare --target-dir=/home/local/data/mysql/dev/full_20260123

# 复制备份到数据目录
./xtrabackup --copy-back --target-dir=/home/local/data/mysql/dev/full_20260123  --datadir=/home/local/data/mysql/dev/restore/
```