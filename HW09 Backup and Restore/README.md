## Создаем ВМ/докер c ПГ.
```
yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub

sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15

```
## Создаем БД, схему и в ней таблицу.
```
postgres=# CREATE DATABASE otus;
CREATE DATABASE
otus=# CREATE SCHEMA tt;
CREATE SCHEMA
otus=# CREATE TABLE tt.test (id SERIAL PRIMARY KEY, text TEXT);
CREATE TABLE
```
## Заполним таблицы автосгенерированными 100 записями.
```
otus=# INSERT INTO tt.test SELECT id, MD5(random()::TEXT)::TEXT FROM generate_series(1, 100) AS id;
INSERT 0 100
```
## Под линукс пользователем Postgres создадим каталог для бэкапов
```
mkdir /backup
chown -R postgres: /backup
```
## Сделаем логический бэкап используя утилиту COPY
```
otus=# COPY tt.test TO '/backup/test';
COPY 100
```
## Восстановим в 2 таблицу данные из бэкапа.
```
otus=# CREATE TABLE tt.test2 (id SERIAL PRIMARY KEY, text TEXT);
CREATE TABLE

otus=# COPY tt.test2 FROM '/backup/test';
COPY 100

```
## Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
```
su - postgres
Password:
postgres@otus-vm2:~$ pg_dump -d otus --compress=9 --table=tt.test --table=tt.test2 -Fc > /backup/backup
_2_tabl.gz
postgres@otus-vm2:~$ pg_restore --list /backup/backup_2_tabl.gz
;
; Archive created at 2024-12-05 06:09:03 UTC
;     dbname: otus
;     TOC Entries: 18
;     Compression: 9
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 14.15 (Ubuntu 14.15-1.pgdg20.04+1)
;     Dumped by pg_dump version: 14.15 (Ubuntu 14.15-1.pgdg20.04+1)
;
;
; Selected TOC Entries:
;
211; 1259 16398 TABLE tt test postgres
213; 1259 16408 TABLE tt test2 postgres
212; 1259 16407 SEQUENCE tt test2_id_seq postgres
3329; 0 0 SEQUENCE OWNED BY tt test2_id_seq postgres
210; 1259 16397 SEQUENCE tt test_id_seq postgres
3330; 0 0 SEQUENCE OWNED BY tt test_id_seq postgres
3174; 2604 16401 DEFAULT tt test id postgres
3175; 2604 16411 DEFAULT tt test2 id postgres
3320; 0 16398 TABLE DATA tt test postgres
3322; 0 16408 TABLE DATA tt test2 postgres
3331; 0 0 SEQUENCE SET tt test2_id_seq postgres
3332; 0 0 SEQUENCE SET tt test_id_seq postgres
3179; 2606 16415 CONSTRAINT tt test2 test2_pkey postgres
3177; 2606 16405 CONSTRAINT tt test test_pkey postgres
```
## Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
```
otus=# CREATE DATABASE otus2;
CREATE DATABASE
otus2=# CREATE SCHEMA tt;
CREATE SCHEMA

postgres@otus-vm2:~$ pg_restore -d otus2 --table=test2 /backup/backup_2_tabl.gz

otus2=# select count(*) from tt.test2;
 count
-------
   100
(1 row)
```