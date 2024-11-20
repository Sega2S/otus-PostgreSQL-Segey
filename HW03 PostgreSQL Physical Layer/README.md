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
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15 
```

## проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q
```
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
```

## остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```
yc-user@otus-vm:~$ sudo -i -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
yc-user@otus-vm:~$ sudo -i -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## создайте новый диск к ВМ размером 10GB
```
 yc compute disk create --name new-disk --type network-hdd --size 10 --description "seconde disk for otus-vm"       
done (10s)
id: fhmqqqa4hoe6hdqti1tu
folder_id: b1g33pe83gmp2574t46a
created_at: "2024-10-30T13:49:34Z"
name: new-disk
description: seconde disk for otus-vm
type_id: network-hdd
zone_id: ru-central1-a
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}
hardware_generation:Brooke logan
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1
```

## добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
```
yc compute instance attach-disk otus-vm --disk-name new-disk --mode rw
yc-user@otus-vm:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE SIZE MOUNTPOINT LABEL
vda            15G
├─vda1          1M
└─vda2 ext4    15G /
vdb            10G
```

## проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
```
yc-user@otus-vm:~$ sudo fdisk /dev/vdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x9c0b1ad0.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): p
Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x9c0b1ad0

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux

-- Отформатируем диск в нужную файловую систему, с помощью утилиты mkfs (файловую систему возьмем EXT4):
sudo mkfs.ext4 /dev/vdb1

mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: f14446f2-0854-42dd-a53f-b554632e3a26
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

-- Смонтируем раздел диска vdb1 в папку /mnt/data, с помощью утилиты mount:
sudo mkdir /mnt/data
sudo mount /dev/vdb1 /mnt/data

yc-user@otus-vm:~$ echo "/dev/disk/by-uuid/f14446f2-0854-42dd-a53f-b554632e3a26 /mnt/data ext4 defaults 0 1" | sudo tee
-a /etc/fstab
/dev/disk/by-uuid/f14446f2-0854-42dd-a53f-b554632e3a26 /mnt/data ext4 defaults 0 1
yc-user@otus-vm:~$ sudo mount -a
```
## перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```
yc-user@otus-vm:~$ sudo reboot
Connection to 89.169.159.77 closed by remote host.
Connection to 89.169.159.77 closed.

yc-user@otus-vm:~$ df -h /mnt/data/
Filesystem      Size  Used Avail Use% Mounted on
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

```
## сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```
yc-user@otus-vm:~$ sudo chown -R postgres. /mnt/data
yc-user@otus-vm:~$ ls -lh /mnt/
total 8.0K
drwxr-xr-x 3 postgres postgres 4.0K Nov 10 12:15 data
```

## перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data

```
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

yc-user@otus-vm:~$ sudo mv /var/lib/postgresql/15/main /mnt/data
```
## попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start

```
yc-user@otus-vm:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
## напишите получилось или нет и почему

```
Не получилось. Потому что данные кластера мы перенесли в другое место
```
## задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
```
yc-user@otus-vm:~$ pg_conftool 15 main show all
cluster_name = '15/main'
data_directory = '/var/lib/postgresql/15/main'
datestyle = 'iso, mdy'
default_text_search_config = 'pg_catalog.english'
dynamic_shared_memory_type = posix
external_pid_file = '/var/run/postgresql/15-main.pid'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Etc/UTC'
max_connections = 100
max_wal_size = 1GB
min_wal_size = 80MB
port = 5432
shared_buffers = 128MB
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
timezone = 'Etc/UTC'
unix_socket_directories = '/var/run/postgresql'

yc-user@otus-vm:~$ sudo -i -u postgres pg_conftool 15 main set data_directory /mnt/data/main
yc-user@otus-vm:~$ sudo -i -u postgres pg_conftool 15 main show data_directory
data_directory = '/mnt/data/main'
```

## напишите что и почему поменял
```
поменял параметр data_directory, тк он указывает каталог для хранения данных.
```
## попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```
yc-user@otus-vm:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Cluster is already running.
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
15  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-15-main.log
```
##напишите получилось или нет и почему
```
Получилось, потому что данные были найдены.
```

## зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
yc-user@otus-vm:~$ sudo -i -u postgres psql -c "select * from test;"

yc-user@otus-vm:~$ sudo -u postgres psql
psql (15.8 (Ubuntu 15.8-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

```
## задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

```
 yc compute instance create --name otus-vm2 --hostname otus-vm2 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
done (40s)
id: fhm06li0sf1tkl70tej0
folder_id: b1g33pe83gmp2574t46a
created_at: "2024-11-11T14:35:21Z"
name: otus-vm2
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm2he0vd5v84u4vf46m
  auto_delete: true
  disk_id: fhm2he0vd5v84u4vf46m
network_interfaces:
  - index: "0"
    mac_address: d0:0d:35:64:0e:3c
    subnet_id: e9br4v65h170ucpv2fl1
    primary_v4_address:
      address: 192.168.0.31
      one_to_one_nat:
        address: 89.169.156.254
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm2.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1


 yc compute instance list
+----------------------+----------+---------------+---------+----------------+--------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+----------+---------------+---------+----------------+--------------+
| fhm06li0sf1tkl70tej0 | otus-vm2 | ru-central1-a | RUNNING | 89.169.156.254 | 192.168.0.31 |
| fhmu9ce0911vo9hos00s | otus-vm  | ru-central1-a | RUNNING | 51.250.71.182  | 192.168.0.15 |
+----------------------+----------+---------------+---------+----------------+--------------+

yc-user@otus-vm2:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
yc compute instance stop otus-vm
 yc compute instance detach-disk otus-vm --disk-name new-disk
done (4s)
id: fhmu9ce0911vo9hos00s
folder_id: b1g33pe83gmp2574t46a
created_at: "2024-10-30T13:16:16Z"
name: otus-vm
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: STOPPED
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmsu2mg9bvov87u5ee8
  auto_delete: true
  disk_id: fhmsu2mg9bvov87u5ee8
network_interfaces:
  - index: "0"
    mac_address: d0:0d:1e:4b:1c:04
    subnet_id: e9br4v65h170ucpv2fl1
    primary_v4_address:
      address: 192.168.0.15
      one_to_one_nat:
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1

yc compute instance attach-disk otus-vm2 --disk-name new-disk
done (14s)
id: fhm06li0sf1tkl70tej0
folder_id: b1g33pe83gmp2574t46a
created_at: "2024-11-11T14:35:21Z"
name: otus-vm2
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm2he0vd5v84u4vf46m
  auto_delete: true
  disk_id: fhm2he0vd5v84u4vf46m
secondary_disks:
  - mode: READ_WRITE
    device_name: fhmqqqa4hoe6hdqti1tu
    disk_id: fhmqqqa4hoe6hdqti1tu
network_interfaces:
  - index: "0"
    mac_address: d0:0d:35:64:0e:3c
    subnet_id: e9br4v65h170ucpv2fl1
    primary_v4_address:
      address: 192.168.0.31
      one_to_one_nat:
        address: 89.169.156.254
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: otus-vm2.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1

yc-user@otus-vm2:~$ echo "/dev/disk/by-uuid/f14446f2-0854-42dd-a53f-b554632e3a26 /mnt/data ext4 defaults 0 1" | sudo tee
 -a /etc/fstab
/dev/disk/by-uuid/f14446f2-0854-42dd-a53f-b554632e3a26 /mnt/data ext4 defaults 0 1

yc-user@otus-vm2:~$ sudo mkdir /mnt/data
yc-user@otus-vm2:~$ sudo chown -R postgres. /mnt/data
yc-user@otus-vm2:~$ sudo mount -a
yc-user@otus-vm2:~$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/vdb1       9.8G   39M  9.2G   1% /mnt/data

sudo -i -u postgres pg_conftool 15 main show data_directory
data_directory = '/var/lib/postgresql/15/main'
sudo systemctl stop postgresql@15-main.service
sudo -i -u postgres pg_conftool 15 main set data_directory /mnt/data/main
sudo systemctl start postgresql@15-main.service
pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
15  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-15-main.log
sudo -i -u postgres psql -c "select * from test;"
 c1
----
 1
(1 row)

```