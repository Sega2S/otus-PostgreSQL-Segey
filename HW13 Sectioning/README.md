Для выполнения домашнего задания взята демо бд «Авиаперевозки» https://postgrespro.ru/education/demodb.
Структура таблиц https://edu.postgrespro.ru/demo-20161013.pdf

Скачиваем архив с БД и распаковываем:
```
wget https://edu.postgrespro.ru/demo-medium.zip
sudo apt install unzip
unzip demo-medium.zip

postgres=# \i  demo-medium-20170815.sql

postgres=# \i  demo-medium-20170815.sql;
SET
SET
SET
SET
SET
SET
SET
SET
psql:demo-medium-20170815.sql:17: ERROR:  database "demo" does not exist
CREATE DATABASE
You are now connected to database "demo" as user "postgres".
SET
SET
SET
SET
SET
SET
SET
SET
CREATE SCHEMA
COMMENT
CREATE EXTENSION
COMMENT
SET
CREATE FUNCTION
CREATE FUNCTION
COMMENT
SET
SET
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE SEQUENCE
ALTER SEQUENCE
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE VIEW
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
CREATE TABLE
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
COMMENT
ALTER TABLE
COPY 9
COPY 104
COPY 1894295
COPY 593433
COPY 65664
COPY 1339
COPY 2360335
COPY 829071
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER TABLE
ALTER DATABASE
ALTER DATABASE
```
```
demo=# \dt+;
                                                 List of relations
  Schema  |      Name       | Type  |  Owner   | Persistence | Access method |  Size   |        Description
----------+-----------------+-------+----------+-------------+---------------+---------+---------------------------
 bookings | aircrafts_data  | table | postgres | permanent   | heap          | 16 kB   | Aircrafts (internal data)
 bookings | airports_data   | table | postgres | permanent   | heap          | 56 kB   | Airports (internal data)
 bookings | boarding_passes | table | postgres | permanent   | heap          | 109 MB  | Boarding passes
 bookings | bookings        | table | postgres | permanent   | heap          | 30 MB   | Bookings
 bookings | flights         | table | postgres | permanent   | heap          | 6368 kB | Flights
 bookings | seats           | table | postgres | permanent   | heap          | 96 kB   | Seats
 bookings | ticket_flights  | table | postgres | permanent   | heap          | 154 MB  | Flight segment
 bookings | tickets         | table | postgres | permanent   | heap          | 109 MB  | Tickets
(8 rows)
```
![table1](https://github.com/Sega2S/otus-PostgreSQL-Segey/blob/master/HW13%20Sectioning/scheme.PNG?raw=true)

Секционируем таблицу bookings (бронирования)
![table1](https://github.com/Sega2S/otus-PostgreSQL-Segey/blob/master/HW13%20Sectioning/table1.PNG?raw=true)
![table2](https://github.com/Sega2S/otus-PostgreSQL-Segey/blob/master/HW13%20Sectioning/table2.PNG?raw=true)
```
demo=# select count(book_ref) from bookings;
 count
--------
 593433
(1 row)
```
У нас долго происходит поиск по брони
```
explain analyze select * from bookings where book_ref='102A61';

                                                       QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_pkey on bookings  (cost=0.42..8.44 rows=1 width=21) (actual time=0.061..0.062 rows=0 loops=1)
   Index Cond: (book_ref = '102A61'::bpchar)
 Planning Time: 0.060 ms
 Execution Time: 0.076 ms
(4 rows)
```
Создадим секционированную таблицу по хэшу
```
demo=# CREATE TABLE bookings_p (
       book_ref     character(6),
       book_date    timestamptz,
       total_amount numeric(10,2),
       CONSTRAINT bookings_p_pkey PRIMARY KEY (book_ref)
   ) PARTITION BY HASH(book_ref);
CREATE TABLE
```
Будем разбивать на 8 партиций
```
demo=# CREATE TABLE bookings_0 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE
demo=# CREATE TABLE bookings_1 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 1);
kings_p FOR VALUES WITH (MODULUS 8, REMAINDER 5);
CREATE TABLE bookings_6 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 6);
CREATE TABLE bookings_7 PARTITION OF bookings_p FOR CREATE TABLE
demo=# CREATE TABLE bookings_2 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 2);
CREATE TABLE
demo=# CREATE TABLE bookings_3 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 3);
VALUES WITH (MODULUS 8, REMAINDER 7);CREATE TABLE
demo=# CREATE TABLE bookings_4 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 4);
CREATE TABLE
demo=# CREATE TABLE bookings_5 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 5);
CREATE TABLE
demo=# CREATE TABLE bookings_6 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 6);
CREATE TABLE
demo=# CREATE TABLE bookings_7 PARTITION OF bookings_p FOR VALUES WITH (MODULUS 8, REMAINDER 7);
```
Получаем следующие
```
demo=# \d+ bookings_p;
                                            Partitioned table "bookings.bookings_p"
    Column    |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
--------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 book_ref     | character(6)             |           | not null |         | extended |             |              |
 book_date    | timestamp with time zone |           |          |         | plain    |             |              |
 total_amount | numeric(10,2)            |           |          |         | main     |             |              |
Partition key: HASH (book_ref)
Indexes:
    "bookings_p_pkey" PRIMARY KEY, btree (book_ref)
Partitions: bookings_0 FOR VALUES WITH (modulus 8, remainder 0),
            bookings_1 FOR VALUES WITH (modulus 8, remainder 1),
            bookings_2 FOR VALUES WITH (modulus 8, remainder 2),
            bookings_3 FOR VALUES WITH (modulus 8, remainder 3),
            bookings_4 FOR VALUES WITH (modulus 8, remainder 4),
            bookings_5 FOR VALUES WITH (modulus 8, remainder 5),
            bookings_6 FOR VALUES WITH (modulus 8, remainder 6),
            bookings_7 FOR VALUES WITH (modulus 8, remainder 7)
```
Заполняем таблицу bookings_p
```
demo=# demo=# INSERT INTO bookings_p SELECT * FROM bookings;
INSERT 0 593433
```
Размер наших секции:
```
demo=# SELECT
demo-#   relname AS table_name,
demo-#   pg_size_pretty(pg_total_relation_size(relid)) AS total,
demo-#   pg_size_pretty(pg_relation_size(relid)) AS internal,
demo-#   pg_size_pretty(pg_table_size(relid) - pg_relation_size(relid)) AS external,
demo-#   pg_size_pretty(pg_indexes_size(relid)) AS indexes
demo-# FROM pg_catalog.pg_statio_user_tables
demo-# WHERE relname like 'bookings_%'
demo-# ORDER BY pg_total_relation_size(relid) DESC;
 table_name |  total  | internal | external | indexes
------------+---------+----------+----------+---------
 bookings_2 | 5480 kB | 3800 kB  | 32 kB    | 1648 kB
 bookings_3 | 5480 kB | 3800 kB  | 32 kB    | 1648 kB
 bookings_0 | 5472 kB | 3792 kB  | 32 kB    | 1648 kB
 bookings_5 | 5464 kB | 3792 kB  | 32 kB    | 1640 kB
 bookings_4 | 5456 kB | 3784 kB  | 32 kB    | 1640 kB
 bookings_1 | 5440 kB | 3776 kB  | 32 kB    | 1632 kB
 bookings_6 | 5432 kB | 3768 kB  | 32 kB    | 1632 kB
 bookings_7 | 5432 kB | 3768 kB  | 32 kB    | 1632 kB
(8 rows)
```
Исправим зависимые таблицы
```
demo=# alter table tickets drop constraint tickets_book_ref_fkey;
ALTER TABLE

demo=# alter table tickets add constraint tickets_book_ref_fkey foreign key (book_ref) references bookings_p (book_ref);
ALTER TABLE

demo=# alter table bookings rename to bookings_old;
ALTER TABLE
demo=# alter table bookings_p rename to bookings;
ALTER TABLE

```
По итогу поиск в секционной таблице проходит быстрее, за счет того что записей в таблице меньше.
До секционирования:
Execution Time: 0.076 ms 
После секционирования:
Execution Time: 0.024 ms
```
explain analyze select * from bookings where book_ref='102A61';
                                                              QUERY PLAN

--------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_0_pkey on bookings_0 bookings  (cost=0.29..8.31 rows=1 width=21) (actual time=0.011..0.011 rows=0 loops=1)
   Index Cond: (book_ref = '102A61'::bpchar)
 Planning Time: 0.208 ms
 Execution Time: 0.024 ms
(4 rows)
```
