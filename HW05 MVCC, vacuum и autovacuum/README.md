Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
```
 yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub

```
Установить на него PostgreSQL 15 с дефолтными настройками
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Создать БД для тестов: выполнить pgbench -i postgres
```
postgres=# create database pgtest
postgres-# ;
CREATE DATABASE

postgres@otus-vm:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.22 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.90 s, vacuum 0.05 s, primary keys 0.26 s).

```
Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
```

postgres@otus-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 230.0 tps, lat 34.307 ms stddev 35.909, 0 failed
progress: 12.0 s, 256.8 tps, lat 31.269 ms stddev 32.669, 0 failed
progress: 18.0 s, 194.3 tps, lat 41.224 ms stddev 40.303, 0 failed
progress: 24.0 s, 203.7 tps, lat 39.196 ms stddev 41.306, 0 failed
progress: 30.0 s, 363.2 tps, lat 22.013 ms stddev 26.276, 0 failed
progress: 36.0 s, 355.5 tps, lat 22.395 ms stddev 29.145, 0 failed
progress: 42.0 s, 215.8 tps, lat 37.132 ms stddev 35.732, 0 failed
progress: 48.0 s, 250.2 tps, lat 31.828 ms stddev 34.916, 0 failed
progress: 54.0 s, 233.5 tps, lat 34.385 ms stddev 44.774, 0 failed
progress: 60.0 s, 329.2 tps, lat 24.232 ms stddev 27.715, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 15801
number of failed transactions: 0 (0.000%)
latency average = 30.390 ms
latency stddev = 34.981 ms
initial connection time = 21.012 ms
tps = 262.807093 (without initial connection time)
```
Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
```
 nano /etc/postgresql/15/main/postgresql.conf

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
Протестировать заново
```
postgres@otus-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 338.5 tps, lat 23.406 ms stddev 27.960, 0 failed
progress: 12.0 s, 394.5 tps, lat 20.295 ms stddev 26.382, 0 failed
progress: 18.0 s, 267.2 tps, lat 29.762 ms stddev 39.154, 0 failed
progress: 24.0 s, 289.0 tps, lat 27.741 ms stddev 34.316, 0 failed
progress: 30.0 s, 290.7 tps, lat 27.503 ms stddev 33.327, 0 failed
progress: 36.0 s, 310.2 tps, lat 25.738 ms stddev 30.992, 0 failed
progress: 42.0 s, 399.3 tps, lat 19.997 ms stddev 27.003, 0 failed
progress: 48.0 s, 259.5 tps, lat 30.812 ms stddev 39.865, 0 failed
progress: 54.0 s, 283.2 tps, lat 28.299 ms stddev 37.695, 0 failed
progress: 60.0 s, 384.2 tps, lat 20.669 ms stddev 29.786, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 19305
number of failed transactions: 0 (0.000%)
latency average = 24.833 ms
latency stddev = 32.548 ms
initial connection time = 24.041 ms
tps = 321.565175 (without initial connection time)
```
Что изменилось и почему?
При первом запуске за 1 минуту работы отработало 15801 транзакции со скорость 262 транзакции в секунду
После применения настроек за 1 минуту работы отработало 19305 транзакции со скорость 321 транзакции в секунду
Пропускная способность базы данных увеличилась почти на 34%. Думаю, что такой результат удалось достичь за счет увеличения параметров wal_buffers, shared_buffers, maintenance_work_mem.

Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```
postgres=# CREATE TABLE test_table(test text);
CREATE TABLE
postgres=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | pgbench_accounts | table | postgres
 public | pgbench_branches | table | postgres
 public | pgbench_history  | table | postgres
 public | pgbench_tellers  | table | postgres
 public | test_table       | table | postgres
(5 rows)

postgres=# INSERT INTO test_table(test) SELECT 'test' FROM generate_series(1,1000000);
INSERT 0 1000000
```
Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_TABLE_size('test_table'));
 pg_size_pretty
----------------
 35 MB
(1 row)
```
5 раз обновить все строчки и добавить к каждой строчке любой символ
```
postgres=# UPDATE test_table SET test = CONCAT(test,'a');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'a');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'a');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'a');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'a');
UPDATE 1000000
```
Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs  WHERE relname = 'test_table';
  relname   | n_live_tup | n_dead_tup | ratio% |       last_autovacuum
------------+------------+------------+--------+------------------------------
 test_table |    1000000 |    1999626 |    199 | 2024-12-17 18:16:29.41092+00
(1 row)
```
Подождать некоторое время, проверяя, пришел ли автовакуум
автовакуум прошел и мертвых строк не осталось
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs  WHERE relname = 'test_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
------------+------------+------------+--------+-------------------------------
 test_table |    1000000 |          0 |      0 | 2024-12-17 18:17:16.163368+00
(1 row)
```
5 раз обновить все строчки и добавить к каждой строчке любой символ
```
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
```
Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_TABLE_size('test_table'));

postgres=# SELECT pg_size_pretty(pg_TABLE_size('test_table'));
 pg_size_pretty
----------------
 211 MB
(1 row)
```
Отключить Автовакуум на конкретной таблице
```
ALTER TABLE test_table SET (autovacuum_enabled = off);
ALTER TABLE
```
10 раз обновить все строчки и добавить к каждой строчке любой символ
```
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
postgres=# UPDATE test_table SET test = CONCAT(test,'s');
UPDATE 1000000
```
Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_TABLE_size('test_table'));
 pg_size_pretty
----------------
 540 MB
(1 row)
```
Объясните полученный результат
```
Автовакуум отключен и при каждом обновлении создается столько же новых записей и они не зачищаются. Следовательно, таблица становится больше. 
```
Не забудьте включить автовакуум)
```
postgres=# ALTER TABLE test_table SET (autovacuum_enabled = on);
ALTER TABLE
```