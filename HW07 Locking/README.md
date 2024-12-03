## Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
```
postgres=# select * from pg_settings where name like 'log_lock_waits' \gx
-[ RECORD 1 ]---+------------------------------------
name            | log_lock_waits
setting         | off
unit            |
category        | Reporting and Logging / What to Log
short_desc      | Logs long lock waits.
extra_desc      |
context         | superuser
vartype         | bool
source          | default
min_val         |
max_val         |
enumvals        |
boot_val        | off
reset_val       | off
sourcefile      |
sourceline      |
pending_restart | f

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

postgres=# select * from pg_settings where name='deadlock_timeout' \gx
-[ RECORD 1 ]---+--------------------------------------------------------------
name            | deadlock_timeout
setting         | 1000
unit            | ms
category        | Lock Management
short_desc      | Sets the time to wait on a lock before checking for deadlock.
extra_desc      |
context         | superuser
vartype         | integer
source          | default
min_val         | 1
max_val         | 2147483647
enumvals        |
boot_val        | 1000
reset_val       | 1000
sourcefile      |
sourceline      |
pending_restart | f

postgres=# alter system set deadlock_timeout to 200;
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show deadlock_timeout ;
 deadlock_timeout
------------------
 200ms
(1 row)

-- session 1
postgres=# CREATE TABLE test_table2 (id SERIAL PRIMARY KEY, text TEXT);
CREATE TABLE

postgres=# INSERT INTO test_table2 SELECT id, MD5(random()::TEXT)::TEXT FROM generate_series(1, 10000) AS id;1
INSERT 0 10000

-- session 2

postgres=# VACUUM FULL test_table2;
VACUUM

2024-12-02 14:28:08.418 UTC [1254] postgres@postgres LOG:  process 1254 still waiting for AccessExclusiveLock on relation 16405 of database 5 after 200.107 ms
2024-12-02 14:28:08.418 UTC [1254] postgres@postgres DETAIL:  Process holding the lock: 1104. Wait queue: 1254.
2024-12-02 14:28:08.418 UTC [1254] postgres@postgres STATEMENT:  VACUUM FULL test_table2;
2024-12-02 14:30:05.111 UTC [1254] postgres@postgres LOG:  process 1254 acquired AccessExclusiveLock on relation 16405 of database 5 after 116892.763 ms
2024-12-02 14:30:05.111 UTC [1254] postgres@postgres STATEMENT:  VACUUM FULL test_table2;

```
## Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
```
postgres=# SELECT pg_relation_filepath('test_table2');
 pg_relation_filepath
----------------------
 base/5/16414
(1 row)

-- session 1
BEGIN
postgres=*# UPDATE test_table2 SET text= '111' where id=2;

-- session 2
BEGIN
postgres=*# UPDATE test_table2 SET text= '222' where id=2;

-- session 3
BEGIN
postgres=*# UPDATE test_table2 SET text= '333' where id=2;

2024-12-02 14:52:51.106 UTC [1597] postgres@postgres LOG:  process 1597 acquired ShareLock on transaction 746 after 8526.111 ms
2024-12-02 14:52:51.106 UTC [1597] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_table2"
2024-12-02 14:52:51.106 UTC [1597] postgres@postgres STATEMENT:  UPDATE test_table2 SET text= '111' where id=2;
2024-12-02 14:53:26.287 UTC [1495] postgres@postgres LOG:  process 1495 still waiting for ShareLock on transaction 748 after 200.100 ms
2024-12-02 14:53:26.287 UTC [1495] postgres@postgres DETAIL:  Process holding the lock: 1597. Wait queue: 1495.
2024-12-02 14:53:26.287 UTC [1495] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test_table2"
2024-12-02 14:53:26.287 UTC [1495] postgres@postgres STATEMENT:  UPDATE test_table2 SET text= '222' where id=2;
2024-12-02 14:53:36.882 UTC [1510] postgres@postgres LOG:  process 1510 still waiting for ExclusiveLock on tuple (0,1) of relation 16405 of database 5 after 200.167 ms
2024-12-02 14:53:36.882 UTC [1510] postgres@postgres DETAIL:  Process holding the lock: 1495. Wait queue: 1510.
2024-12-02 14:53:36.882 UTC [1510] postgres@postgres STATEMENT:  UPDATE test_table2 SET text= '333' where id=2;


postgres=# select pg_blocking_pids(1597);
 pg_blocking_pids
------------------
 {}
(1 row)

postgres=# select pg_blocking_pids(1495);
 pg_blocking_pids
------------------
 {1597}
(1 row)

postgres=# select pg_blocking_pids(1510);
 pg_blocking_pids
------------------
 {1495}
(1 row)

postgres=# select 'test_table2'::regclass::int;
 int4
-------
 16405
(1 row)

postgres=# select 16405::regclass;
  regclass
-------------
 test_table2
(1 row)

postgres=# select locktype,database,relation,page,tuple,transactionid,virtualtransaction,pid,mode,granted,fastpath from pg_locks where pid in (1597,1495,1510)
postgres-# ;
   locktype    | database | relation | page | tuple | transactionid | virtualtransaction | pid  |       mode       | granted | fastpath
---------------+----------+----------+------+-------+---------------+--------------------+------+------------------+---------+----------
 relation      |        5 |    16411 |      |       |               | 7/32               | 1510 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 7/32               | 1510 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 7/32               | 1510 | ExclusiveLock    | t       | t
 relation      |        5 |    16411 |      |       |               | 6/59               | 1597 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 6/59               | 1597 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 6/59               | 1597 | ExclusiveLock    | t       | t
 relation      |        5 |    16411 |      |       |               | 5/115              | 1495 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 5/115              | 1495 | RowExclusiveLock | t       | t
 virtualxid    |          |          |      |       |               | 5/115              | 1495 | ExclusiveLock    | t       | t
 transactionid |          |          |      |       |           748 | 5/115              | 1495 | ShareLock        | f       | f
 tuple         |        5 |    16405 |    0 |     1 |               | 7/32               | 1510 | ExclusiveLock    | f       | f
 tuple         |        5 |    16405 |    0 |     1 |               | 5/115              | 1495 | ExclusiveLock    | t       | f
 transactionid |          |          |      |       |           750 | 7/32               | 1510 | ExclusiveLock    | t       | f
 transactionid |          |          |      |       |           749 | 5/115              | 1495 | ExclusiveLock    | t       | f
 transactionid |          |          |      |       |           748 | 6/59               | 1597 | ExclusiveLock    | t       | f
(15 rows)

1) Транзакция обновила данные tuple и получила эксклюзивную блокировку для целевой таблицы и ключа
 relation      |        5 |    16411 |      |       |               | 6/59               | 1597 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 6/59               | 1597 | RowExclusiveLock | t       | t

Экслюзивная блокировка транзакции самой на себя xid и vxid
 virtualxid    |          |          |      |       |               | 6/59               | 1597 | ExclusiveLock    | t       | t

Исключительная блокировка номера транзакции
transactionid |          |          |      |       |           748 | 6/59               | 1597 | ExclusiveLock    | t       | f

2) Получила эксклюзивную блокировку для целевой таблицы и ключа
 relation      |        5 |    16411 |      |       |               | 5/115              | 1495 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 5/115              | 1495 | RowExclusiveLock | t       | t

Экслюзивная блокировка транзакции самой на себя xid и vxid
 virtualxid    |          |          |      |       |               | 5/115              | 1495 | ExclusiveLock    | t       | t

Исключительная блокировка номера транзакции
 transactionid |          |          |      |       |           749 | 5/115              | 1495 | ExclusiveLock    | t       | f

Эксклюзивная блокировка версии строки для обновления 
 tuple         |        5 |    16405 |    0 |     1 |               | 5/115              | 1495 | ExclusiveLock    | t       | f

Установки ShareLock раздельной блокировки на заблокировавщую транзакцию строку
 transactionid |          |          |      |       |           748 | 5/115              | 1495 | ShareLock        | f       | f

3) Экслюзивная блокировка транзакции самой на себя xid и vxid
virtualxid    |          |          |      |       |               | 7/32               | 1510 | ExclusiveLock    | t       | t

Исключительная блокировка номера транзакции
transactionid |          |          |      |       |           750 | 7/32               | 1510 | ExclusiveLock    | t       | f

Получила эксклюзивную блокировку для целевой таблицы и ключа
 relation      |        5 |    16411 |      |       |               | 7/32               | 1510 | RowExclusiveLock | t       | t
 relation      |        5 |    16405 |      |       |               | 7/32               | 1510 | RowExclusiveLock | t       | t

Эксклюзивная блокировка версии строки для обновления не удалась  
 tuple         |        5 |    16405 |    0 |     1 |               | 7/32               | 1510 | ExclusiveLock    | f       | f

postgres=#  SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'test_table2'::regclass order by pid;
 locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1495 | {1597}
 tuple    | ExclusiveLock    | t       | 1495 | {1597}
 relation | RowExclusiveLock | t       | 1510 | {1495}
 tuple    | ExclusiveLock    | f       | 1510 | {1495}
 relation | RowExclusiveLock | t       | 1597 | {}
(5 rows)
```
## Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
```

-- session 1
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_table2 SET text = '11111' where id=1;
UPDATE 1

-- session 2

postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_table2 SET text = '22222' where id=2;
UPDATE 1

-- session 3

postgres=# BEGIN;
BEGIN
postgres=*# UPDATE test_table2 SET text = '33333' where id=3;
UPDATE 1

-- session 1

UPDATE test_table2 SET text = '11111' where id=2;

-- session 2

UPDATE test_table2 SET text = '22222' where id=3;

-- session 3

postgres=*# UPDATE test_table2 SET text = '33333' where id=1;
ERROR:  deadlock detected
DETAIL:  Process 2806 waits for ShareLock on transaction 757; blocked by process 2705.
Process 2705 waits for ShareLock on transaction 758; blocked by process 2755.
Process 2755 waits for ShareLock on transaction 759; blocked by process 2806.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (83,50) in relation "test_table2"

2024-12-02 16:35:34.948 UTC [2755] postgres@postgres LOG:  process 2755 still waiting for ShareLock on transaction 755 after 200.147 ms
2024-12-02 16:35:34.948 UTC [2755] postgres@postgres DETAIL:  Process holding the lock: 2705. Wait queue: 2755.
2024-12-02 16:35:34.948 UTC [2755] postgres@postgres CONTEXT:  while updating tuple (83,48) in relation "test_table2"
2024-12-02 16:35:34.948 UTC [2755] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '22222' where id=2;
2024-12-02 16:37:11.846 UTC [910] LOG:  checkpoint starting: time
2024-12-02 16:37:12.180 UTC [910] LOG:  checkpoint complete: wrote 4 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.303 s, sync=0.020 s, total=0.335 s; sync files=3, longest=0.015 s, average=0.007 s; distance=19 kB, estimate=849 kB
2024-12-02 16:37:55.855 UTC [2755] postgres@postgres LOG:  process 2755 acquired ShareLock on transaction 755 after 141107.247 ms
2024-12-02 16:37:55.855 UTC [2755] postgres@postgres CONTEXT:  while updating tuple (83,48) in relation "test_table2"
2024-12-02 16:37:55.855 UTC [2755] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '22222' where id=2;
2024-12-02 16:38:19.835 UTC [2806] postgres@postgres WARNING:  there is no transaction in progress
2024-12-02 16:41:09.012 UTC [2705] postgres@postgres LOG:  process 2705 still waiting for ShareLock on transaction 758 after 200.668 ms
2024-12-02 16:41:09.012 UTC [2705] postgres@postgres DETAIL:  Process holding the lock: 2755. Wait queue: 2705.
2024-12-02 16:41:09.012 UTC [2705] postgres@postgres CONTEXT:  while updating tuple (83,48) in relation "test_table2"
2024-12-02 16:41:09.012 UTC [2705] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '11111' where id=2;
2024-12-02 16:41:15.628 UTC [2755] postgres@postgres LOG:  process 2755 still waiting for ShareLock on transaction 759 after 200.159 ms
2024-12-02 16:41:15.628 UTC [2755] postgres@postgres DETAIL:  Process holding the lock: 2806. Wait queue: 2755.
2024-12-02 16:41:15.628 UTC [2755] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test_table2"
2024-12-02 16:41:15.628 UTC [2755] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '22222' where id=3;
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres LOG:  process 2806 detected deadlock while waiting for ShareLock on transaction 757 after 200.179 ms
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres DETAIL:  Process holding the lock: 2705. Wait queue: .
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres CONTEXT:  while updating tuple (83,50) in relation "test_table2"
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '33333' where id=1;
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres ERROR:  deadlock detected
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres DETAIL:  Process 2806 waits for ShareLock on transaction 757; blocked by process 2705.
        Process 2705 waits for ShareLock on transaction 758; blocked by process 2755.
        Process 2755 waits for ShareLock on transaction 759; blocked by process 2806.
        Process 2806: UPDATE test_table2 SET text = '33333' where id=1;
        Process 2705: UPDATE test_table2 SET text = '11111' where id=2;
        Process 2755: UPDATE test_table2 SET text = '22222' where id=3;
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres HINT:  See server log for query details.
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres CONTEXT:  while updating tuple (83,50) in relation "test_table2"
2024-12-02 16:41:24.212 UTC [2806] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '33333' where id=1;
2024-12-02 16:41:24.212 UTC [2755] postgres@postgres LOG:  process 2755 acquired ShareLock on transaction 759 after 8784.237 ms
2024-12-02 16:41:24.212 UTC [2755] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test_table2"
2024-12-02 16:41:24.212 UTC [2755] postgres@postgres STATEMENT:  UPDATE test_table2 SET text = '22222' where id=3;
2024-12-02 16:42:11.280 UTC [910] LOG:  checkpoint starting: time
2024-12-02 16:42:11.734 UTC [910] LOG:  checkpoint complete: wrote 4 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.355 s, sync=0.005 s, total=0.454 s; sync files=3, longest=0.004 s, average=0.002 s; distance=19 kB, estimate=766 kB
 
 Можно разобраться изучая журнал сообщении, но сложно
 Отстреленным является Process 2806: UPDATE test_table2 SET text = '33333' where id=1;
 
```
## Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
```
Возможна, если одна команда будет обновлять строки таблицы в прямом порядке, а другая - в обратном.
```
