```
+----------------------+----------+---------------+---------+----------------+--------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+----------+---------------+---------+----------------+--------------+
| fhmgcc18c8nss7862rdi | otus-vm1 | ru-central1-a | RUNNING | 84.201.159.43  | 192.168.0.23 |
| fhmo0nvq2kdktra69004 | otus-vm  | ru-central1-a | STOPPED |                | 192.168.0.6  |
| fhmsesskusug3m08h0g8 | otus-vm2 | ru-central1-a | RUNNING | 84.201.129.255 | 192.168.0.19 |
| fhmtcoruqtc3aodjkdna | otus-vm3 | ru-central1-a | RUNNING | 158.160.42.255 | 192.168.0.35 |
+----------------------+----------+---------------+---------+----------------+--------------+
```
1 вм
host    all             postgres        84.201.129.255/32          scram-sha-256
host    all             postgres        158.160.42.255/32          scram-sha-256
2 вм
host    all             postgres        84.201.159.43/32        scram-sha-256
host    all             postgres        158.160.42.255/32       scram-sha-256

## На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
```
postgres=# CREATE TABLE test (id int, txt char(15));
CREATE TABLE
postgres=# CREATE TABLE test2 (id int, txt char(15));
CREATE TABLE

```
## Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
```
postgres=# INSERT INTO test SELECT generate_series(1,10) as id, md5(random()::text)::char(15) as TEXT;
INSERT 0 10

postgres=# CREATE PUBLICATION test_pub FOR table test;
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION

Исправляем ошибку
postgres=# ALTER SYSTEM SET wal_level to 'logical';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET listen_addresses TO '*';
ALTER SYSTEM

```
## На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
```
postgres=# SHOW wal_level;
 wal_level

postgres=# CREATE TABLE test (id int, txt char(15));
CREATE TABLE
postgres=# CREATE TABLE test2 (id int, txt char(15));
CREATE TABLE

```
## Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
```
postgres=# INSERT INTO test2 SELECT generate_series(1,10) as id, md5(random()::text)::char(15) as TEXT;
INSERT 0 10

postgres=# CREATE PUBLICATION test_pub2 FOR table test2;
CREATE PUBLICATION

На 2й ВМ создаю подписку на публикацию таблицы test 1й ВМ:
postgres=# CREATE SUBSCRIPTION test_sub  connection 'host=84.201.159.43 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub with (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION

postgres=# \dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication
----------+----------+---------+-------------
 test_sub | postgres | t       | {test_pub}
(1 row)

Проверяем:
postgres=# select * from pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16400
subname               | test_sub
pid                   | 6229
relid                 |
received_lsn          | 0/155D8D0
last_msg_send_time    | 2024-12-05 13:52:33.44747+00
last_msg_receipt_time | 2024-12-05 13:52:33.429925+00
latest_end_lsn        | 0/155D8D0
latest_end_time       | 2024-12-05 13:52:33.44747+00

На 1й ВМ подписываюсь на публикацию таблицы test2 2-й ВМ
postgres=# CREATE SUBSCRIPTION test_sub  connection 'host=84.201.129.255 port=5432 user=postgres password=postgres dbname=postgres' publication test_pub2 with (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION

postgres=# select * from pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16400
subname               | test_sub
pid                   | 6681
relid                 |
received_lsn          | 0/1560578
last_msg_send_time    | 2024-12-05 14:01:27.715385+00
last_msg_receipt_time | 2024-12-05 14:01:27.733024+00
latest_end_lsn        | 0/1560578
latest_end_time       | 2024-12-05 14:01:27.715385+00

```
## 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
```
postgres=# CREATE TABLE test (id int, txt char(15));
CREATE TABLE
postgres=# CREATE TABLE test2 (id int, txt char(15));
CREATE TABLE
postgres=# CREATE SUBSCRIPTION test3_sub1  connection 'host=84.201.159.43 port=5432 user=postgres passw
ord=postgres dbname=postgres' publication test_pub with (copy_data = true);
NOTICE:  created replication slot "test3_sub1" on publisher
CREATE SUBSCRIPTION
postgres=# CREATE SUBSCRIPTION test3_sub2  connection 'host=84.201.129.255 port=5432 user=postgres pass
word=postgres dbname=postgres' publication test_pub2 with (copy_data = true);
NOTICE:  created replication slot "test3_sub2" on publisher
CREATE SUBSCRIPTION

postgres=# select * from pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16394
subname               | test3_sub1
pid                   | 6020
relid                 |
received_lsn          | 0/1564FD0
last_msg_send_time    | 2024-12-05 14:09:20.623232+00
last_msg_receipt_time | 2024-12-05 14:09:20.624298+00
latest_end_lsn        | 0/1564FD0
latest_end_time       | 2024-12-05 14:09:20.623232+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16395
subname               | test3_sub2
pid                   | 6023
relid                 |
received_lsn          | 0/15605E8
last_msg_send_time    | 2024-12-05 14:09:33.817026+00
last_msg_receipt_time | 2024-12-05 14:09:33.828152+00
latest_end_lsn        | 0/15605E8
latest_end_time       | 2024-12-05 14:09:33.817026+00
```