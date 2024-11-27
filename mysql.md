

```sql

-- json_table
-- table1
-----------------
id | code | name
-----------------
1  | A    | zhangsan
2  | A    | lisi
3  | B    | wangwu
------------------

-- table2
----------------
id | sid | val
---------------
 1 | 1   | 0.3
 2 | 2   | 0.5
 3 | 1   | 0.4
 4 | 3   | 0.1
 5 | 2   | 0.2
--------------

-- 通过table1.code分组, 获取table2.val的和
-----------------
code | name    |  val
 A   | zhs,lsi | 1.4
 B   | wangw   | 0.3
-----------------

> select 
>   a.code,group_concat(a.name),sum(b.val) 
> from table1 a 
> join table2 b on a.id=b.sid 
> group by a.code;

-- 是json_arrayagg和json_table的子查询
> select a.code, a.name, sum(b.val) from (
>  select * from (
>   select json_arrayagg(id) ids,code,group_concat(name) name from table1 group by code limit 0,10
>   ) a, json_table(a.ids, '$[*]' columns (
>     id bigint path '$'
>   )) as b
> ) a join table2 b on a.id=b.sid group by a.code;

-------------------
> select json_table('[1,2,3]', columns (
>     id bigint path '$'
> ));
----
1
2
3
---

```