---
title: MySQL解析JSON格式数据
tags: [MySQL,JSON]
author: Yc-Ma
show_author_profile: true
key: 2021-12-22-MySQL解析JSON格式数据
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## MySQL解析JSON格式数据
- 输出多行
```
SELECT
	*
FROM
	JSON_TABLE('[{"c1":1},{"c1":2}]',
	'$[*]' COLUMNS( c1 INT PATH '$.c1' ERROR ON ERROR )) as jt;
```
- 输出多行JSON对象：输出列表每个元素不解析
```
SELECT
	*
FROM
	JSON_TABLE('[{"c1":1},{"c1":2}]',
	'$[*]' COLUMNS( c1 JSON PATH '$' ERROR ON ERROR )) as jt;
```
- 输出多行：根据不同的字段类型选举输出
```
SELECT *
 FROM
   JSON_TABLE(
     '[{"a":"3"},{"a":2},{"b":1},{"a":0},{"a":[1,2]}]',
     '$[*]'
     COLUMNS(
       rowid FOR ORDINALITY,
       ac VARCHAR(100) PATH '$.a' DEFAULT '111' ON EMPTY DEFAULT '999' ON ERROR,
       aj JSON PATH '$.a' DEFAULT '{"x": 333}' ON EMPTY,
       bx INT EXISTS PATH '$.b'
     )
) AS tt;
```
- LIST-MAP对象解析输出
```
SELECT *
 FROM
   JSON_TABLE(
     '[{"x":2,"y":"8"},{"x":"3","y":"7"},{"x":"4","y":6}]',
     '$[*]' COLUMNS(
       xval VARCHAR(100) PATH '$.x',
       yval VARCHAR(100) PATH '$.y'
     )
   ) AS  jt1;
```
- 解析指定索引下表的对象
```
SELECT *
 FROM
   JSON_TABLE(
     '[{"x":2,"y":"8"},{"x":"3","y":"7"},{"x":"4","y":6}]',
     '$[1]' COLUMNS(
       xval VARCHAR(100) PATH '$.x',
       yval VARCHAR(100) PATH '$.y'
     )
   ) AS  jt1;
```
- 解析嵌套的LIST为多行
```
SELECT *
 FROM
   JSON_TABLE(
     '[ {"a": 1, "b": [11,111]}, {"a": 2, "b": [22,222]}, {"a":3}]',
     '$[*]' COLUMNS(
             a INT PATH '$.a',
             NESTED PATH '$.b[*]' COLUMNS (b INT PATH '$')
            )
    ) AS jt
 WHERE b IS NOT NULL;
```
- 解析嵌套的LIST为多行：嵌套多行输出
```
SELECT *
 FROM
  JSON_TABLE(
     '[{"a": 1, "b": [11,111]}, {"a": 2, "b": [22,222]}]',
     '$[*]' COLUMNS(
        a INT PATH '$.a',
        NESTED PATH '$.b[*]' COLUMNS (b1 INT PATH '$'),
        NESTED PATH '$.b[*]' COLUMNS (b2 INT PATH '$')
     )
 ) AS jt;
```
- 更复杂的嵌套多行输出
```
SELECT *
 FROM
   JSON_TABLE(
     '[{"a": "a_val","b": [{"c": "c_val", "l": [1,2]}]},{"a": "a_val","b": [{"c": "c_val","l": [11]}, {"c": "c_val", "l": [22]}]}]',
     '$[*]' COLUMNS(
       top_ord FOR ORDINALITY,
       apath VARCHAR(10) PATH '$.a',
       NESTED PATH '$.b[*]' COLUMNS (
        bpath VARCHAR(10) PATH '$.c',
         ord FOR ORDINALITY,
         NESTED PATH '$.l[*]' COLUMNS (lpath varchar(10) PATH '$')
         )
     )
 ) as jt;
```
- 将表中的JSON解析后多行一起输出
```
CREATE TABLE t1 (c1 INT, c2 CHAR(1), c3 JSON);
INSERT INTO t1 () VALUES
	(1, 'z', JSON_OBJECT('a', 23, 'b', 27, 'c', 1)),
	(1, 'y', JSON_OBJECT('a', 44, 'b', 22, 'c', 11)),
	(2, 'x', JSON_OBJECT('b', 1, 'c', 15)),
	(3, 'w', JSON_OBJECT('a', 5, 'b', 6, 'c', 7)),
	(5, 'v', JSON_OBJECT('a', 123, 'c', 1111));
SELECT * FROM t1
SELECT c1, c2, JSON_EXTRACT(c3, '$.*')
FROM t1 AS m
JOIN
JSON_TABLE(
  m.c3,
  '$.*'
  COLUMNS(
    at VARCHAR(10) PATH '$.a' DEFAULT '1' ON EMPTY,
    bt VARCHAR(10) PATH '$.b' DEFAULT '2' ON EMPTY,
    ct VARCHAR(10) PATH '$.c' DEFAULT '3' ON EMPTY
  )
) AS tt
ON m.c1 > tt.at;
```
- 获取数组中指定元素
```
SELECT JSON_EXTRACT(JSON_ARRAY(1,2,3), '$[0]')
SELECT JSON_EXTRACT('[1,2,3]', '$[0]')
```
- 在MySQL中将曾用名JSON列表转为多行输出
```
SELECT previous_name,tt.*
 FROM table AS a
 JOIN
 JSON_TABLE(
   a.previous_name,
   '$[*]'
   COLUMNS(previous_nm JSON PATH '$' ERROR ON ERROR )
 ) AS tt
 WHERE a.previous_name IS NOT NULL AND a.previous_name<>''
```


