

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
./xtrabackup --prepare --target-dir=/data/backup/fullmysql
# increment
./xtrabackup --prepare \
--target-dir=/data/backup/fullmysql \
--apply-log-only

./xtrabackup --prepare \
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
# backup
./xtrabackup  --user=ro --password='xx@2025.RD' --host=192.168.1.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/full_20260123 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4

rsync -avz full_20260123 root@192.168.1.5:/home/local/data/mysql/dev/

./xtrabackup  --user=ro --password='xx@2025.RD' --host=192.168.1.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/ince_1 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4 --incremental-basedir=/home/data/mysqlbackup/dev/full_20260123

./xtrabackup  --user=ro --password='x@2025.RD' --host=192.168.6.4 --port=31603 --backup --target-dir=/home/data/mysqlbackup/dev/ince_2 --no-lock  --datadir=/home/rancher/mysql/2/data --parallel=4 --incremental-basedir=/home/data/mysqlbackup/dev/ince_1

rsync -avz ince_1 192.168.1.5:/home/local/data/mysql/dev/
rsync -avz ince_2 192.168.1.5:/home/local/data/mysql/dev/

# restore
./xtrabackup --prepare --apply-log-only --target-dir=/home/local/data/mysql/dev/full_20260123

./xtrabackup --prepare --apply-log-only --target-dir=/home/local/data/mysql/dev/full_20260123 --incremental-dir=/home/local/data/mysql/dev/ince_1

./xtrabackup --prepare  --target-dir=/home/local/data/mysql/dev/full_20260123 --incremental-dir=/home/local/data/mysql/dev/ince_2

./xtrabackup --copy-back --target-dir=/home/local/data/mysql/dev/full_20260123  --datadir=/home/local/data/mysql/dev/restore/
```