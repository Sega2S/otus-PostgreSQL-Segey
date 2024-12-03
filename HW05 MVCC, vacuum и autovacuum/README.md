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
```
Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
```
pgbench -i test_db

pgbench -c8 -P 6 -T 60 -U postgres test_db
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 379.2 tps, lat 20.877 ms stddev 24.309, 0 failed
progress: 12.0 s, 422.2 tps, lat 18.986 ms stddev 23.771, 0 failed
progress: 18.0 s, 314.2 tps, lat 25.358 ms stddev 28.640, 0 failed
progress: 24.0 s, 309.7 tps, lat 25.040 ms stddev 30.086, 0 failed
progress: 30.0 s, 108.8 tps, lat 73.631 ms stddev 59.886, 0 failed
progress: 36.0 s, 218.8 tps, lat 37.589 ms stddev 40.330, 0 failed
progress: 42.0 s, 274.2 tps, lat 29.158 ms stddev 32.245, 0 failed
progress: 48.0 s, 347.5 tps, lat 22.931 ms stddev 27.288, 0 failed
progress: 54.0 s, 368.2 tps, lat 21.667 ms stddev 25.627, 0 failed
progress: 60.0 s, 229.3 tps, lat 34.969 ms stddev 35.268, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 17840
number of failed transactions: 0 (0.000%)
latency average = 26.868 ms
latency stddev = 32.572 ms
initial connection time = 23.999 ms
tps = 297.275890 (without initial connection time)
```
Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
```
```
Протестировать заново
```
```
Что изменилось и почему?
```
```
Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```
```
Посмотреть размер файла с таблицей
```
```
5 раз обновить все строчки и добавить к каждой строчке любой символ
```
```
Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
```
Подождать некоторое время, проверяя, пришел ли автовакуум
```
```
5 раз обновить все строчки и добавить к каждой строчке любой символ
```
```
Посмотреть размер файла с таблицей
```
```
Отключить Автовакуум на конкретной таблице
```
```
10 раз обновить все строчки и добавить к каждой строчке любой символ
```
```
Посмотреть размер файла с таблицей
```
```
Объясните полученный результат
```
```
Не забудьте включить автовакуум)