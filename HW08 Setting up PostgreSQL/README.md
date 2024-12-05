## развернуть виртуальную машину любым удобным способом
```
yc compute instance create --name otus-vm2 --hostname otus-vm2 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub

```
## поставить на неё PostgreSQL 15 любым способом
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
postgres=# create database test_db
postgres-# ;
CREATE DATABASE

sudo -i -u postgres bash
pgbench -i test_db;
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
done in 1.07 s (drop tables 0.00 s, create tables 0.03 s, client-side generate 0.65 s, vacuum 0.09 s, primary keys 0.30 s).

pgbench -c 40 -j 2 -P 10 -T 60 test_db;
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 150.4 tps, lat 259.061 ms stddev 306.214, 0 failed
progress: 20.0 s, 275.9 tps, lat 145.681 ms stddev 172.752, 0 failed
progress: 30.0 s, 348.4 tps, lat 113.387 ms stddev 125.900, 0 failed
progress: 40.0 s, 176.5 tps, lat 227.757 ms stddev 289.901, 0 failed
progress: 50.0 s, 249.6 tps, lat 159.481 ms stddev 176.006, 0 failed
progress: 60.0 s, 247.1 tps, lat 160.395 ms stddev 171.180, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 40
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 14519
number of failed transactions: 0 (0.000%)
latency average = 165.730 ms
latency stddev = 206.223 ms
initial connection time = 82.995 ms
tps = 240.272565 (without initial connection time)
```
## настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
```

max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
work_mem = 5242kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
synchronous_commit = off
fsync = off
wal_level = minimal
checkpoint_timeout = 30min
full_page_writes = off
max_wal_senders = 0
```
## нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
```
pgbench -c 40 -j 2 -P 10 -T 60 test_db;
pgbench (15.10 (Ubuntu 15.10-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 1624.7 tps, lat 24.310 ms stddev 19.761, 0 failed
progress: 20.0 s, 1632.2 tps, lat 24.402 ms stddev 20.256, 0 failed
progress: 30.0 s, 1618.3 tps, lat 24.624 ms stddev 20.085, 0 failed
progress: 40.0 s, 1651.0 tps, lat 24.129 ms stddev 19.471, 0 failed
progress: 50.0 s, 1632.0 tps, lat 24.419 ms stddev 19.694, 0 failed
progress: 60.0 s, 1615.7 tps, lat 24.674 ms stddev 20.921, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 40
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 97778
number of failed transactions: 0 (0.000%)
latency average = 24.443 ms
latency stddev = 20.058 ms
initial connection time = 73.661 ms
tps = 1629.136338 (without initial connection time

```
## написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
```
               sourcefile                |             name             |                setting                 | applied
-----------------------------------------+------------------------------+----------------------------------------+---------
 /etc/postgresql/15/main/postgresql.conf | data_directory               | /var/lib/postgresql/15/main            | t
 /etc/postgresql/15/main/postgresql.conf | hba_file                     | /etc/postgresql/15/main/pg_hba.conf    | t
 /etc/postgresql/15/main/postgresql.conf | ident_file                   | /etc/postgresql/15/main/pg_ident.conf  | t
 /etc/postgresql/15/main/postgresql.conf | external_pid_file            | /var/run/postgresql/15-main.pid        | t
 /etc/postgresql/15/main/postgresql.conf | port                         | 5432                                   | t
 /etc/postgresql/15/main/postgresql.conf | max_connections              | 100                                    | f
 /etc/postgresql/15/main/postgresql.conf | unix_socket_directories      | /var/run/postgresql                    | t
 /etc/postgresql/15/main/postgresql.conf | ssl                          | on                                     | t
 /etc/postgresql/15/main/postgresql.conf | ssl_cert_file                | /etc/ssl/certs/ssl-cert-snakeoil.pem   | t
 /etc/postgresql/15/main/postgresql.conf | ssl_key_file                 | /etc/ssl/private/ssl-cert-snakeoil.key | t
 /etc/postgresql/15/main/postgresql.conf | shared_buffers               | 128MB                                  | f
 /etc/postgresql/15/main/postgresql.conf | dynamic_shared_memory_type   | posix                                  | t
 /etc/postgresql/15/main/postgresql.conf | max_wal_size                 | 1GB                                    | f
 /etc/postgresql/15/main/postgresql.conf | min_wal_size                 | 80MB                                   | f
 /etc/postgresql/15/main/postgresql.conf | log_line_prefix              | %m [%p] %q%u@%d                        | t
 /etc/postgresql/15/main/postgresql.conf | log_timezone                 | Etc/UTC                                | t
 /etc/postgresql/15/main/postgresql.conf | cluster_name                 | 15/main                                | t
 /etc/postgresql/15/main/postgresql.conf | datestyle                    | iso, mdy                               | t
 /etc/postgresql/15/main/postgresql.conf | timezone                     | Etc/UTC                                | t
 /etc/postgresql/15/main/postgresql.conf | lc_messages                  | en_US.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_monetary                  | en_US.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_numeric                   | en_US.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | lc_time                      | en_US.UTF-8                            | t
 /etc/postgresql/15/main/postgresql.conf | default_text_search_config   | pg_catalog.english                     | t
 /etc/postgresql/15/main/postgresql.conf | max_connections              | 100                                    | t
 /etc/postgresql/15/main/postgresql.conf | shared_buffers               | 1GB                                    | t
 /etc/postgresql/15/main/postgresql.conf | effective_cache_size         | 3GB                                    | t
 /etc/postgresql/15/main/postgresql.conf | maintenance_work_mem         | 256MB                                  | t
 /etc/postgresql/15/main/postgresql.conf | checkpoint_completion_target | 0.9                                    | t
 /etc/postgresql/15/main/postgresql.conf | wal_buffers                  | 16MB                                   | t
 /etc/postgresql/15/main/postgresql.conf | default_statistics_target    | 100                                    | t
 /etc/postgresql/15/main/postgresql.conf | random_page_cost             | 4                                      | t
 /etc/postgresql/15/main/postgresql.conf | work_mem                     | 5242kB                                 | t
 /etc/postgresql/15/main/postgresql.conf | huge_pages                   | off                                    | t
 /etc/postgresql/15/main/postgresql.conf | min_wal_size                 | 1GB                                    | t
 /etc/postgresql/15/main/postgresql.conf | max_wal_size                 | 4GB                                    | t
 /etc/postgresql/15/main/postgresql.conf | synchronous_commit           | off                                    | t
 /etc/postgresql/15/main/postgresql.conf | fsync                        | off                                    | t
 /etc/postgresql/15/main/postgresql.conf | wal_level                    | minimal                                | t
 /etc/postgresql/15/main/postgresql.conf | checkpoint_timeout           | 30min                                  | t
 /etc/postgresql/15/main/postgresql.conf | full_page_writes             | off                                    | t
 /etc/postgresql/15/main/postgresql.conf | max_wal_senders              | 0                                      | t
(42 rows)

Указынные параметры были предложены pgtune
max_connections = 100 - максимальное число одновременных подключений.
shared_buffers = 1GB - объём памяти, который будет использовать сервер баз данных для буферов в разделяемой памяти. 
effective_cache_size = 3GB -  памяти, доступная для кэширования диска.
maintenance_work_mem = 256MB -  - размер памяти, выделяемой служебным процессам.Увеличение может ускорить построение индексов и процесс очистки - vacuum.
checkpoint_completion_target = 0.9 - доля времени между контрольными точками для завершения контрольной точки в PostgreSQL.
wal_buffers = 16MB - Объём общей памяти, используемой для данных WAL, которые ещё не были записаны на диск. 
default_statistics_target = 100 -  целевое значение статистики по умолчанию для столбцов таблицы.
random_page_cost = 4 - Задаёт оценку планировщиком стоимости непоследовательно извлекаемой страницы диска.
work_mem = 5242kB - Задаёт базовый максимальный объём памяти, который будет использоваться во внутренних операциях при обработке запросов.
huge_pages = off
min_wal_size = 1GB - задаёт минимальный размер журнального сегмента WAL.
max_wal_size = 4GB - задаёт максимальный размер журнального сегмента WAL.
synchronous_commit = off - определяет, когда транзакции считаются зафиксированными и в какой момент клиент получает подтверждение об этом. off - Транзакции считаются зафиксированными сразу после записи в журнал WAL, без ожидания записи на диск. Это может значительно повысить производительность, но с риском потери данных при сбое.
fsync = off - параметр, который отвечает за сброс данных из кэша на диск при завершении транзакций. Если установить значение fsync=off, то данные не будут записываться на дисковые накопители сразу после завершения операций. Это может существенно повысить скорость операций insert и update, но есть риск повредить базу, если произойдёт сбой (неожиданное отключение питания, сбой ОС, сбой дисковой подсистемы).
wal_level = minimal - определяет, сколько информации записывается в WAL (журнал операций). minimal- Записывает только информацию, необходимую для восстановления после сбоя или немедленного завершения работы. 
checkpoint_timeout = 30min - Максимальное время между автоматическими контрольными точками в WAL.
full_page_writes = off - при включённом значении заставляет сервер записывать в WAL всё содержимое каждой страницы при первом изменении этой страницы после контрольной точки. В нашем случае для максимальной производительности, мы его отключаем.
max_wal_senders = 0 - ожидаемое число реплик.

было
tps = 240.272565 (without initial connection time)
стало 
tps = 1629.136338 (without initial connection time

Увеличение производительности достигнуто за счет отключения надежности записи в WAL, синхронизации WAL с диском.
```
