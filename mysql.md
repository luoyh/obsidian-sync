

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