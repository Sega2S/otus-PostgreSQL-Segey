## Создаем сетевую инфраструктуру и саму ВМ:
```
yc vpc network create --name otus-net --description "otus-net"
yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
cat ~/.ssh/yc_key.pub
yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
```

## Узнал ip вм
```
yc compute instance get otus-vm 
```

## Подключение к ВМ по EXTERNAL IP
```
ssh -i yc_key yc-user@51.250.91.63
```

## Устанавливаем posgres
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14 
```

## Открыть прослушивание всех адресов
```
sudo nano /etc/postgresql/14/main/postgresql.conf
sudo nano /etc/postgresql/14/main/pg_hba.conf
```
## Меняем пароль пользователя
```
sudo -u postgres psql
alter user postgres password 'postgres';
```

## При изменении файлов .conf нужен ребут
```
sudo pg_ctlcluster 14 main restart
```

## Выключаем  auto commit
```
sudo -u postgres psql
ostgres=# \echo :AUTOCOMMIT
on
ostgres=# \set AUTOCOMMIT off
ostgres=# \echo :AUTOCOMMIT
off
```

## в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```sql
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```
## посмотреть текущий уровень изоляции 
```sql
postgres=# show transaction isolation level
postgres-# ;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
## начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
## в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
```sql
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
## сделать select from persons во второй сессии
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```    
## видите ли вы новую запись и если да то почему?
```
Не видим, потомоу что AUTOCOMMIT отключен и уровень изоляции по умолчанию read commited.
```
## завершить первую транзакцию - commit;
```sql
postgres=*# commit;
COMMIT
```    
## делать select from persons во второй сессии
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
## видите ли вы новую запись и если да то почему?
```
Видим, потому что транзакция по добавлению новых завершена.
```    
## завершите транзакцию во второй сессии
```sql
postgres=*# commit;
COMMIT
```
## начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
## в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
```sql
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

## сделать select* from persons во второй сессии*
```sql
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# select* from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
## видите ли вы новую запись и если да то почему?
```
Нет, так как первая транзакция еще не завершена. 
```
## завершить первую транзакцию - commit;
```sql
postgres=*# commit;
COMMIT
```
## сделать select from persons во второй сессии
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```    
## видите ли вы новую запись и если да то почему?
```
Не видим, так как теперь у нас не завершена транзакция в первой сессии с уровнем изоляции repeatable read. Данные от первой появились, но для транзакции во второй сессии они не видны.
```
## завершить вторую транзакцию
```sql
postgres=*# commit;
COMMIT
```    
## сделать select * from persons во второй сессии
```sql
postgres=# select * from persons ;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```    
## видите ли вы новую запись и если да то почему?
```
Видим, так как начась новая транзакция во второй сессии и мы видим все предыдущие завершенные транзакции.
```
