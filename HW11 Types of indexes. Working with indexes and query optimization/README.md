postgres=# CREATE DATABASE Test_db;
CREATE DATABASE

postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
test_db=#  CREATE TABLE test as
test_db-# SELECT
test_db-# generate_series as id
test_db-# ,generate_series::text || (random() * 10)::text as txt
test_db-# ,(array['Ivan','Sergey','Nina','Inna','Roman'])[floor(random() * 5 + 1)] as array
test_db-# ,random() > 0.5 as random_boolean
test_db-# FROM generate_series(1, 100000);

test_db=# select * from test limit 10;
 id |         txt          | array  | random_boolean
----+----------------------+--------+----------------
  1 | 13.632038510018658   | Nina   | f
  2 | 25.6066804797943215  | Ivan   | t
  3 | 39.986557296920635   | Roman  | f
  4 | 41.4027213186304555  | Nina   | f
  5 | 56.895945980770755   | Roman  | f
  6 | 60.41616568840828316 | Roman  | t
  7 | 72.1185576590363153  | Nina   | f
  8 | 83.888809845253709   | Ivan   | f
  9 | 94.672808512605715   | Nina   | f
 10 | 106.081953618313911  | Sergey | f
(10 rows)

Статистика по таблице
test_db=# analyze test;
ANALYZE
test_db=# select attname, correlation from pg_stats where tablename = 'test';
    attname     | correlation
----------------+-------------
 id             |           1
 txt            |  0.82117134
 array          |  0.20567356
 random_boolean |    0.502827
(4 rows)

## Создать индекс к какой-либо из таблиц вашей БД
```
план выполнения запроса без индекса по полю id:
explain (analyze, buffers) select id from test;

 Seq Scan on test  (cost=0.00..1828.00 rows=100000 width=4) (actual time=0.007..9.966 rows=100000 loops=1)
   Buffers: shared hit=828
 Planning:
   Buffers: shared hit=3
 Planning Time: 0.059 ms
 Execution Time: 13.824 ms
(6 rows)

Результат выдал последовательное сканирование по таблице(seq scan)
Cтоимость получения первой строки = 0
shared_hit - количество страниц, которые необходимо прочитать 828
Стоимость получения всех строк 1828
Ширина получения массива данных width = 4
Время получения actual_time = первой строки (0.007), всех строк (9.966)
Строк отдано rows = 100000

test_db=# CREATE INDEX "id" on test (id);
CREATE INDEX

```
## Прислать текстом результат команды explain, в которой используется данный индекс
```
EXPLAIN (analyze, buffers) select id from test where id = 10;

 Index Only Scan using id on test  (cost=0.29..4.31 rows=1 width=4) (actual time=0.035..0.035 rows=1 loops=1)
   Index Cond: (id = 10)
   Heap Fetches: 0
   Buffers: shared hit=1 read=2
 Planning:
   Buffers: shared hit=32 read=1
 Planning Time: 0.321 ms
 Execution Time: 0.059 ms
(8 rows)

Используется Index Only Scan, в рузультате чего уменьшается стоимость запроса и время выполнения. 
```
## Реализовать индекс для полнотекстового поиска
```
Полнотекстный поиск обычно применяют по полям типа text и json

test_db=# EXPLAIN (analyze, buffers) select * from test where test.array like '%Ivan%';

 Seq Scan on test  (cost=0.00..2078.00 rows=19790 width=33) (actual time=0.008..13.011 rows=19891 loops=1)
   Filter: ("array" ~~ '%Ivan%'::text)
   Rows Removed by Filter: 80109
   Buffers: shared hit=828
 Planning:
   Buffers: shared hit=34
 Planning Time: 0.197 ms
 Execution Time: 13.832 ms
(8 rows)

Медленное последовательное сканирование  Seq Scan и высокая стоимость запроса

Для индекса добавим столбец типа tsvector, заполняем данными и создаем индес на поле.

test_db=# ALTER TABLE test ADD COLUMN array_tsvector tsvector;
ALTER TABLE

test_db=# UPDATE test SET array_tsvector = to_tsvector('english', test.array);
UPDATE 100000

test_db=# CREATE INDEX array_tsvector ON test USING GIN (array_tsvector);
CREATE INDEX

test_db=# EXPLAIN (analyze, buffers) select * from test where array_tsvector @@ to_tsquery('Ivan');

 Bitmap Heap Scan on test  (cost=192.44..7340.60 rows=20153 width=50) (actual time=2.077..6.958 rows=19891 loops=1)
   Recheck Cond: (array_tsvector @@ to_tsquery('Ivan'::text))
   Heap Blocks: exact=1031
   Buffers: shared hit=1037
   ->  Bitmap Index Scan on array_tsvector  (cost=0.00..187.40 rows=20153 width=0) (actual time=1.937..1.938 rows=19891 loops=1)
         Index Cond: (array_tsvector @@ to_tsquery('Ivan'::text))
         Buffers: shared hit=6
 Planning:
   Buffers: shared hit=141 dirtied=4
 Planning Time: 1.477 ms
 Execution Time: 7.775 ms
(11 rows)

Сканирование идет по Bitmap Heap Scan и время выполнекния уменишлост с 13.832 ms до 7.775 ms

```
## Реализовать индекс на часть таблицы или индекс на поле с функцией
```
Создадим индекс с условием id < 1000:

test_db=# CREATE INDEX "id_1000" on test(id) where id < 1000;
CREATE INDEX

Размер индекса с условием гораздо меньше

test_db=# SELECT pg_size_pretty(pg_total_relation_size('id_1000'));
 pg_size_pretty
----------------
 40 kB
(1 row)

test_db=# SELECT pg_size_pretty(pg_total_relation_size('id'));
 pg_size_pretty
----------------
 4408 kB
(1 row)
```
## Создать индекс на несколько полей
```
 Bitmap Heap Scan on test  (cost=434.00..2419.58 rows=10206 width=50) (actual time=0.927..2.690 rows=10024 loops=1)
   Recheck Cond: (id < 20000)
   Filter: random_boolean
   Heap Blocks: exact=206
   ->  Bitmap Index Scan on id_random_boolean  (cost=0.00..431.45 rows=10206 width=0) (actual time=0.901..0.901 rows=10024 loops=1)
         Index Cond: ((id < 20000) AND (random_boolean = true))
 Planning Time: 0.139 ms
 Execution Time: 3.025 ms
(8 rows)


test_db=# test_db=# CREATE INDEX "id_random_boolean" on test(id, random_boolean);
CREATE INDEX


Bitmap Heap Scan on test  (cost=66.78..1933.19 rows=1513 width=50) (actual time=0.182..0.433 rows=1461 loops=1)
   Recheck Cond: (id < 3000)
   Filter: random_boolean
   Heap Blocks: exact=31
   ->  Bitmap Index Scan on id_random_boolean  (cost=0.00..66.40 rows=1513 width=0) (actual time=0.172..0.172 rows=1461 loops=1)
         Index Cond: ((id < 3000) AND (random_boolean = true))
 Planning Time: 0.210 ms
 Execution Time: 0.492 ms
(8 rows)

После создания индекса, стоимость запроса стало ниже и время выполнения уменьшилось.
```
