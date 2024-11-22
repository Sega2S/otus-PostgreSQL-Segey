создайте новый кластер PostgresSQL 14
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15 
```
зайдите в созданный кластер под пользователем postgres
```
sudo -u postgres psql
```
создайте новую базу данных testdb
```
postgres=# create database testdb;
CREATE DATABASE
```
зайдите в созданную базу данных под пользователем postgres
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
создайте новую схему testnm
```
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# \dn
  List of schemas
  Name  |  Owner
--------+----------
 public | postgres
 testnm | postgres
(2 rows)
```
создайте новую таблицу t1 с одной колонкой c1 типа integer
```
testdb=# create table t1 ( c1 integer );
CREATE TABLE
testdb=# \dt t1
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```
вставьте строку со значением c1=1
```
testdb=# insert into t1 values (1);
INSERT 0 1
testdb=# select * from t1;
 c1
----
  1
(1 row)
```
создайте новую роль readonly
```
testdb=# create role readonly;
CREATE ROLE
```
дайте новой роли право на подключение к базе данных testdb
```
testdb=# grant CONNECT ON DATABASE testdb TO readonly ;
GRANT
```
дайте новой роли право на использование схемы testnm
```
testdb=# grant USAGE ON SCHEMA testnm TO readonly ;
GRANT
```
дайте новой роли право на select для всех таблиц схемы testnm
```
testdb=# grant SELECT ON ALL TABLES IN SCHEMA testnm TO readonly ;
GRANT
```
создайте пользователя testread с паролем test123
```
testdb=# create user testread with password 'test123';
CREATE ROLE
```
дайте роль readonly пользователю testread
```
testdb=# grant readonly TO testread ;
GRANT ROLE
```
зайдите под пользователем testread в базу данных testdb
```
yc-user@otus-vm:~$ psql -h localhost -d testdb -U testread;
Password for user testread:
psql (14.15 (Ubuntu 14.15-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```
сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
напишите что именно произошло в тексте домашнего задания
у вас есть идеи почему? ведь права то дали?
посмотрите на список таблиц
подсказка в шпаргалке под пунктом 20
а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
```
Не получилось, потому что таблица находится в другой схеме. При неявном указании схемы, по умолчанию встает public.

testdb=> \dt public.t1
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=> \dn+
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         |
 testnm | postgres | postgres=UC/postgres+|
        |          | readonly=U/postgres  |
(2 rows)
```
вернитесь в базу данных testdb под пользователем postgres
```
yc-user@otus-vm:~$ sudo -u postgres psql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
удалите таблицу t1
```
testdb=# drop table t1 ;
DROP TABLE
```
создайте ее заново но уже с явным указанием имени схемы testnm
```
testdb=# create table testnm.t1 ( c1 integer );
CREATE TABLE
testdb=# \d testnm.t1
                 Table "testnm.t1"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 c1     | integer |           |          |
```
вставьте строку со значением c1=1
```
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
```
зайдите под пользователем testread в базу данных testdb
```
yc-user@otus-vm:~$ psql -h localhost -d testdb -U testread;
Password for user testread:
psql (14.15 (Ubuntu 14.15-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```
сделайте select * from testnm.t1;
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
получилось?
есть идеи почему? если нет - смотрите шпаргалку
```
таблица создана после предоставления прав на select. Права только у владельца.
```
как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
```
изменить привелегии по умолчанию для схемы testnm
testdb=# alter default privileges IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
```
сделайте select * from testnm.t1;
получилось?
есть идеи почему? если нет - смотрите шпаргалку
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1

Таблица создана после изменения схемы.
Есди создать новую таблицу по схеме testnm, то все получится
testdb=# create table testnm.t2 ( c1 integer);
CREATE TABLE
testdb=# \dp testnm.t2
                                Access privileges
 Schema | Name | Type  |     Access privileges     | Column privileges | Policies
--------+------+-------+---------------------------+-------------------+----------
 testnm | t2   | table | postgres=arwdDxt/postgres+|                   |
        |      |       | readonly=r/postgres       |                   |
(1 row)

testdb=# select * from testnm.t2;
 c1
----
(0 rows)

Заново выдадим права для роли readoly на чтение всех таблиц
testdb=# grant SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
сделайте select * from testnm.t1;
получилось?
ура!
```
Получилось

testdb=> \dp testnm.t1;
                                Access privileges
 Schema | Name | Type  |     Access privileges     | Column privileges | Policies
--------+------+-------+---------------------------+-------------------+----------
 testnm | t1   | table | postgres=arwdDxt/postgres+|                   |
        |      |       | readonly=r/postgres       |                   |
(1 row)
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

```
теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
```
testdb=# create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
по умолчанию даются права на схему default всем пользователям.
testdb=# select * from pg_catalog.pg_namespace;
  oid  |      nspname       | nspowner |                   nspacl
-------+--------------------+----------+--------------------------------------------
    99 | pg_toast           |       10 |
    11 | pg_catalog         |       10 | {postgres=UC/postgres,=U/postgres}
  2200 | public             |       10 | {postgres=UC/postgres,=UC/postgres}
 13394 | information_schema |       10 | {postgres=UC/postgres,=U/postgres}
 16385 | testnm             |       10 | {postgres=UC/postgres,readonly=U/postgres}
(5 rows)

```
есть идеи как убрать эти права? если нет - смотрите шпаргалку
если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
```
Отобрать права на схему public
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE
после пользователь не сможет создавать в public
```
теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
расскажите что получилось и почему
```
Потому что мы отобрали права на созздание в схеме public {postgres=UC/postgres,=U/postgres}
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
        ^
```