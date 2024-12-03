## Настройте выполнение контрольной точки раз в 30 секунд.
```
sudo -u postgres psql
psql (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
Type "help" for help.

postgres=#  show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

postgres=# show checkpoint_completion_target ;
 checkpoint_completion_target
------------------------------
 0.9
(1 row)

postgres=# alter system set checkpoint_timeout TO '30s';
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show checkpoint_timeout ;
 checkpoint_timeout
--------------------
 30s
(1 row)

```
## 10 минут c помощью утилиты pgbench подавайте нагрузку.
```
postgres=# create database test_db;
CREATE DATABASE

~$ sudo -i -u postgres bash
postgres@otus-vm:~$ pgbench -i testdb
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
done in 0.69 s (drop tables 0.00 s, create tables 0.10 s, client-side generate 0.36 s, vacuum 0.05 s, primary keys 0.18 s).

postgres=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/3F36FE0          | 0/3F37018
(1 row)

pgbench -P 60 -T 600 test_db
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 290.0 tps, lat 3.447 ms stddev 6.336, 0 failed
progress: 120.0 s, 314.3 tps, lat 3.181 ms stddev 6.126, 0 failed
progress: 180.0 s, 271.6 tps, lat 3.682 ms stddev 6.581, 0 failed
progress: 240.0 s, 323.1 tps, lat 3.094 ms stddev 5.937, 0 failed
progress: 300.0 s, 326.8 tps, lat 3.060 ms stddev 5.981, 0 failed
progress: 360.0 s, 319.4 tps, lat 3.130 ms stddev 6.127, 0 failed
progress: 420.0 s, 310.1 tps, lat 3.224 ms stddev 6.208, 0 failed
progress: 480.0 s, 293.8 tps, lat 3.403 ms stddev 6.444, 0 failed
progress: 540.0 s, 304.6 tps, lat 3.283 ms stddev 6.241, 0 failed
progress: 600.0 s, 306.9 tps, lat 3.258 ms stddev 6.159, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 183632
number of failed transactions: 0 (0.000%)
latency average = 3.267 ms
latency stddev = 6.210 ms
initial connection time = 3.592 ms
tps = 306.054853 (without initial connection time)

```
## Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```
postgres=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 0/197FDB50         | 0/197FDB50
(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 131
checkpoints_req       | 1
checkpoint_write_time | 782093
checkpoint_sync_time  | 599
buffers_checkpoint    | 42064
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 5418
buffers_backend_fsync | 0
buffers_alloc         | 8555
stats_reset           | 2024-11-27 17:09:11.586901+00

postgres=# select pg_size_pretty('0/197FDB50'::pg_lsn - '0/3F36FE0'::pg_lsn);
 pg_size_pretty
----------------
 345 MB
(1 row)

в среднем на одну контрольную точку (21 конторльных точек за 10 минут)

select pg_size_pretty(('0/197FDB50'::pg_lsn - '0/3F36FE0'::pg_lsn) / 21 );
 pg_size_pretty
----------------
 16 MB
(1 row)

```
## Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 142
checkpoints_req       | 1
checkpoint_write_time | 782093
checkpoint_sync_time  | 599
buffers_checkpoint    | 42064
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 5418
buffers_backend_fsync | 0
buffers_alloc         | 8579
stats_reset           | 2024-11-27 17:09:11.586901+00

checkpoints_req - остался такой же все чекпоинты уложились в checkpoints_timed.

postgres=# show max_wal_size;
 max_wal_size
--------------
 1GB
(1 row)

Все по расписанию, потому что данных не достаточно чтобы сработал чекпоинт от max_wal_size
```
## Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
```
Отключим
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show synchronous_commit ;
 synchronous_commit
--------------------
 off
(1 row)

postgres=# show wal_writer_delay ;
 wal_writer_delay
------------------
 200ms
(1 row)

postgres=# show fsync;
 fsync
-------
 on
(1 row)

postgres=# show wal_sync_method;
 wal_sync_method
-----------------
 fdatasync
(1 row)

postgres=# show data_checksums;
 data_checksums
----------------
 off
(1 row)

postgres@otus-vm:~$ pgbench -P 60 -T 600 test_db
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 1849.6 tps, lat 0.540 ms stddev 0.225, 0 failed
progress: 120.0 s, 1851.6 tps, lat 0.540 ms stddev 0.126, 0 failed
progress: 180.0 s, 1845.1 tps, lat 0.542 ms stddev 0.166, 0 failed
progress: 240.0 s, 1865.0 tps, lat 0.536 ms stddev 0.194, 0 failed
progress: 300.0 s, 1843.0 tps, lat 0.542 ms stddev 0.265, 0 failed
progress: 360.0 s, 1823.7 tps, lat 0.548 ms stddev 0.082, 0 failed
progress: 420.0 s, 1842.7 tps, lat 0.542 ms stddev 0.079, 0 failed
progress: 480.0 s, 1855.4 tps, lat 0.539 ms stddev 0.091, 0 failed
progress: 540.0 s, 1817.2 tps, lat 0.550 ms stddev 0.187, 0 failed
progress: 600.0 s, 1834.5 tps, lat 0.545 ms stddev 0.159, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1105676
number of failed transactions: 0 (0.000%)
latency average = 0.542 ms
latency stddev = 0.168 ms
initial connection time = 3.843 ms
tps = 1842.803054 (without initial connection time)

Вызов fdatasync() подобен fsync(), но не записывает изменившиеся метаданные, если эти метаданные не нужны для последующего получения данных. Например, изменения st_atime или st_mtime (время последнего доступа и последнего изменения, соответственно; см. stat(2)) не нужно записывать, так как они ненужны для чтения самих данных. С другой стороны, при изменении размера файла (st_size, изменяется, например, ftruncate(2)) запись метаданных будет нужна.

Целью создания fdatasync() является сокращение обменов с диском для приложений, которым не нужна синхронизация метаданных с диском.

Получили большую производительность. С выключеным параметром synchronous_commit не гарантируется что wal записан на диск и не вызвается fdatasync() который имеет большие накладные расходы.

```
## Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
```
pg_dropcluster 15 main --stop
pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file

pg_createcluster 15 main --start -- -k
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions -k
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

show data_checksums ;
 data_checksums
----------------
 on
(1 row)

postgres=# CREATE TABLE test_table (id SERIAL PRIMARY KEY, text TEXT);
CREATE TABLE
postgres=# INSERT INTO test_table SELECT id, MD5(random()::TEXT)::TEXT FROM generate_series(1, 10000) AS id;
INSERT 0 10000

postgres=# select * from test_table limit 10;
 id |               text
----+----------------------------------
  1 | 374752b41678fd32458d240a98449d89
  2 | c3273475b4d7e8642a2af46066174b4f
  3 | 56b581c6ca3784ba18cc60a7854052bb
  4 | b1e73040c2f8ac43fecc5ce079c08930
  5 | ef43587de88b142437af73c44d1f6c96
  6 | e3680110882936c37b68579e804922d6
  7 | b116f11175fa8d1ad9f0a8d37646534f
  8 | b0ce6eac09c3d679209746cbeb1e6aa7
  9 | cbc68a75bbe975978ea2f8186bdccf5a
 10 | 5e32bf276d05f8a4903ede8f41f9839f
(10 rows)

postgres=# SELECT pg_relation_filepath('test_table');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)


postgres@otus-vm:~$  sudo systemctl stop postgresql@15-main

postgres@otus-vm:~$ dd if=/dev/random of=/var/lib/postgresql/15/main/base/5/16389 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.0231049 s, 0.3 kB/s

sudo systemctl start postgresql@15-main

postgres=# select * from test_table limit 10;
WARNING:  page verification failed, calculated checksum 59127 but expected 58058
ERROR:  invalid page in block 0 of relation base/5/16389

postgres=# alter system set ignore_checksum_failure to true ;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from test_table limit 10;
WARNING:  page verification failed, calculated checksum 35250 but expected 57870
 id |               text
----+----------------------------------
  1 | 374752b41678fd32458d240a98449d89
  2 | c3273475b4d7e8642a2af46066174b4f
  3 | 56b581c6ca3784ba18cc60a7854052bb
  4 | b1e73040c2f8ac43fecc5ce079c08930
  5 | ef43587de88b142437af73c44d1f6c96
  6 | e3680110882936c37b68579e804922d6
  7 | b116f11175fa8d1ad9f0a8d37646534f
  8 | b0ce6eac09c3d679209746cbeb1e6aa7
  9 | cbc68a75bbe975978ea2f8186bdccf5a
 10 | 5e32bf276d05f8a4903ede8f41f9839f
(10 rows)

Пересоздадим файлы через VACUUM FULL
postgres=# VACUUM FULL test_table ;
VACUUM

postgres=# select * from test_table limit 10;
 id |               text
----+----------------------------------
  1 | 374752b41678fd32458d240a98449d89
  2 | c3273475b4d7e8642a2af46066174b4f
  3 | 56b581c6ca3784ba18cc60a7854052bb
  4 | b1e73040c2f8ac43fecc5ce079c08930
  5 | ef43587de88b142437af73c44d1f6c96
  6 | e3680110882936c37b68579e804922d6
  7 | b116f11175fa8d1ad9f0a8d37646534f
  8 | b0ce6eac09c3d679209746cbeb1e6aa7
  9 | cbc68a75bbe975978ea2f8186bdccf5a
 10 | 5e32bf276d05f8a4903ede8f41f9839f
(10 rows)
```