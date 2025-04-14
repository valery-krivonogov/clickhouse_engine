#  Выполнение домашнего задания:

## Условия ДЗ:<br>
+ По заданным описаниям таблиц и вставки данных определить используемый движок 
+ Заполнить пропуски, запустить код
+ Сравнить полученный вывод и результат из условия

Сервер развертывается на одной ноде: clickhouse-server <br>


### CollapsingMergeTree
In this engine, you can define a sign column and ask the database to delete stall rows with sign=-1 and keep the new row with sign=1.

Для примера sign_del используется в качестве колонки с признаком удаления
```
-- DDL
CREATE TABLE my_database.tbl1
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    sign_del Int8,
    Version UInt8
)
ENGINE = CollapsingMergeTree(sign_del)
ORDER BY UserID;

-- DML
INSERT INTO my_database.tbl1 VALUES (4324182021466249494, 5, 146,  1, 1);
INSERT INTO my_database.tbl1 VALUES (4324182021466249494, 5, 146, -1, 1);
INSERT INTO my_database.tbl1 VALUES (4324182021466249494, 6, 185,  1, 2);

SELECT *
FROM my_database.tbl1

Query id: 7ea2891c-73a1-4c52-bbe3-15b1df8419eb

   ┌──────────────UserID─┬─PageViews─┬─Duration─┬─sign_del─┬─Version─┐
1. │ 4324182021466249494 │         6 │      185 │        1 │       2 │
2. │ 4324182021466249494 │         5 │      146 │       -1 │       1 │
3. │ 4324182021466249494 │         5 │      146 │        1 │       1 │
   └─────────────────────┴───────────┴──────────┴──────────┴─────────┘

SELECT *
FROM my_database.tbl1
FINAL

Query id: 612b558a-33b1-401e-a561-ca9e297f2875

   ┌──────────────UserID─┬─PageViews─┬─Duration─┬─sign_del─┬─Version─┐
1. │ 4324182021466249494 │         6 │      185 │        1 │       2 │
   └─────────────────────┴───────────┴──────────┴──────────┴─────────┘

1 row in set. Elapsed: 0.008 sec.
```
Запись, помеченная к удалению, отсутствует в таблице

### EmbeddedRocksDB Engine
This engine allows integrating ClickHouse with RocksDB

```
-- DDL
CREATE TABLE my_database.tbl2
(
    key UInt32,
    value UInt32
)
ENGINE = EmbeddedRocksDB
PRIMARY KEY key;

-- DML
INSERT INTO my_database.tbl2 Values(1,1),(1,2),(2,1);

SELECT *
FROM my_database.tbl2

Query id: b7861b77-929b-41d4-9cfc-4f8a1c26329f

   ┌─key─┬─value─┐
1. │   1 │     2 │
2. │   2 │     1 │
   └─────┴───────┘

2 rows in set. Elapsed: 0.006 sec.
```

### ReplaingMergeTree
In this engine, rows with equal order keys are replaced by the last row. Consider the below engine:<br/>

```
-- DDL
CREATE TABLE my_database.tbl3
(
    `id` Int32,
    `status` String,
    `price` String,
    `comment` String
)
ENGINE = ReplacingMergeTree
PRIMARY KEY (id)
ORDER BY (id, status);

-- DML
INSERT INTO my_database.tbl3 VALUES (23, 'success', '1000', 'Confirmed');
INSERT INTO my_database.tbl3 VALUES (23, 'success', '2000', 'Cancelled'); 

SELECT *
FROM my_database.tbl3
WHERE id = 23

Query id: 3b63e298-c8bf-476c-84af-0ee254b63ce3

   ┌─id─┬─status──┬─price─┬─comment───┐
1. │ 23 │ success │ 2000  │ Cancelled │
2. │ 23 │ success │ 1000  │ Confirmed │
   └────┴─────────┴───────┴───────────┘

2 rows in set. Elapsed: 0.007 sec.


SELECT *
FROM my_database.tbl3
FINAL
WHERE id = 23

Query id: c6886fd7-759e-4221-8c6a-645777cc8ccd

   ┌─id─┬─status──┬─price─┬─comment───┐
1. │ 23 │ success │ 2000  │ Cancelled │
   └────┴─────────┴───────┴───────────┘

1 row in set. Elapsed: 0.005 sec.
```

### AggreragatingMergeTree

This engine helps you reduce the response time of heavy, fixed analytics queries by calculating them in writing time. That will end up decreasing in database load in query time too.

```
-- DDL
CREATE TABLE my_database.tbl4
(   CounterID UInt8,
    StartDate Date,
    UserID UInt64
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

CREATE TABLE my_database.tbl5
(   CounterID UInt8,
    StartDate Date,
    UserID AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

-- DML
INSERT INTO my_database.tbl4 VALUES(0, '2019-11-11', 1);
INSERT INTO my_database.tbl4 VALUES(1, '2019-11-12', 1);

INSERT INTO my_database.tbl5
SELECT CounterID, StartDate, uniqState(UserID)
FROM my_database.tbl4
GROUP BY CounterID, StartDate;

-- error
INSERT INTO my_database.tbl5 FORMAT Values
Query id: f331eb0d-b56e-4a72-9c14-8d140ffb3941
Ok.
Error on processing query: Code: 53. DB::Exception: Cannot convert UInt64 to AggregateFunction(u(uniq, UInt64): While executing ValuesBlockInputFormat: data for INSERT was parsed from query.(Y (TYPE_MISMATCH) (version 25.3.2.39 (official build))

-- select 

SELECT uniqMerge(UserID) AS state
FROM my_database.tbl5
GROUP BY
    CounterID,
    StartDate

Query id: 8e418ede-67e6-4f94-857c-1f73b19961e4

   ┌─state─┐
1. │     1 │
2. │     1 │
   └───────┘

2 rows in set. Elapsed: 0.006 sec.
```

### CollapsingMergeTree
sign используется в качестве колонки с признаком удаления

```
-- DDL
CREATE TABLE  my_database.tbl6
(
    `id` Int32,
    `status` String,
    `price` String,
    `comment` String,
    `sign` Int8
)
ENGINE =CollapsingMergeTree(sign)
ORDER BY id;

-- DML
INSERT INTO my_database.tbl6 VALUES (23, 'success', '1000', 'Confirmed', 1);
INSERT INTO my_database.tbl6 VALUES (23, 'success', '1000', 'Confirmed', -1), (23, 'success', '2000', 'Cancelled', 1);

SELECT *
FROM my_database.tbl6

Query id: e74bf9dd-3215-45e6-9b56-a4b321695f75

   ┌─id─┬─status──┬─price─┬─comment───┬─sign─┐
1. │ 23 │ success │ 1000  │ Confirmed │   -1 │
2. │ 23 │ success │ 2000  │ Cancelled │    1 │
3. │ 23 │ success │ 1000  │ Confirmed │    1 │
   └────┴─────────┴───────┴───────────┴──────┘

3 rows in set. Elapsed: 0.005 sec.

SELECT *
FROM my_database.tbl6
FINAL

Query id: fa07cb8c-1de9-4849-b8e5-b947f6dc2c3a

   ┌─id─┬─status──┬─price─┬─comment───┬─sign─┐
1. │ 23 │ success │ 2000  │ Cancelled │    1 │
   └────┴─────────┴───────┴───────────┴──────┘

1 row in set. Elapsed: 0.005 sec.
```

## Результат выполнения
Результат выполнения контрольных запросов соответствует условиям задания