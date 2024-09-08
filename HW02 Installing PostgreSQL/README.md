## создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
ВМ с Ubuntu 20.04 создана на прошлом задании, просто подключаемся к ней через ssh
```
PS C:\Users\Sergey\> ssh -i yc_key yc-user@89.169.131.200
```

## поставить на нем Docker Engine
```
yc-user@otus-vm:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
# Executing docker install script, commit: 0d6f72e671ba87f7aa4c6991646a1a5b9f9dae84
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq ca-certificates curl >/dev/null
+ sh -c install -m 0755 -d /etc/apt/keyrings
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" -o /etc/apt/keyrings/docker.asc
+ sh -c chmod a+r /etc/apt/keyrings/docker.asc
+ sh -c echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-ce-rootless-extras docker-buildx-plugin >/dev/null
+ sh -c docker version
Client: Docker Engine - Community
 Version:           27.2.0
 API version:       1.47
 Go version:        go1.21.13
 Git commit:        3ab4256
 Built:             Tue Aug 27 14:15:09 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          27.2.0
  API version:      1.47 (minimum version 1.24)
  Go version:       go1.21.13
  Git commit:       3ab5c7d
  Built:            Tue Aug 27 14:15:09 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.7.21
  GitCommit:        472731909fa34bd7bc9c087e4c27943f9835f111
 runc:
  Version:          1.1.13
  GitCommit:        v1.1.13-0-g58aa920
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================
```

-- Создаем docker-сеть:
```
yc-user@otus-vm:~$ sudo docker network create pg-net
888cdde9e3ba0231327486a8899ee69a52f9d7e3b8978d009060379cff851edc
```

### сделать каталог /var/lib/postgres
### развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
```
-- подключаем созданную сеть к контейнеру сервера Postgres:
yc-user@otus-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
a2318d6c47ec: Pull complete
293ff58ffc95: Pull complete
e03fd0ed521a: Pull complete
618b85f70713: Pull complete
fefee7186940: Pull complete
6bd68dbb7324: Pull complete
2fd2333708a2: Pull complete
a9193c797654: Pull complete
9db671ef13b2: Pull complete
229510061cf7: Pull complete
5d0d6d069b83: Pull complete
c31960c64c87: Pull complete
1c0a1dca29f6: Pull complete
de96e3ff4e81: Pull complete
Digest: sha256:e600c23d564a5fc7824093c89b4c25dacb1cdd5de64e9e7ced0c630cfbcc1890
Status: Downloaded newer image for postgres:15
92b147e4663a20c30a15fd3b20d23efc67769b8ca1acea410be37bdffb00717c
```

### развернуть контейнер с клиентом postgres
```
-- Запускаем отдельный контейнер с клиентом в общей сети с БД: 
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres

postgres=# \conninfo
You are connected to database "postgres" as user "postgres" on host "pg-server" (address "172.18.0.2") at port "5432".
```

### подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
### подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов ЯО/места установки докера
```
psql -p 5432 -U postgres -h 89.169.131.200 -d otus -W
otus=# create table students(id serial, first_name text, second_name text);
CREATE TABLE
otus=# insert into students(first_name, second_name) values('ivanov','ivan'),('petrov','petr');
INSERT 0 2
otus=# select * from students;
 id | first_name | second_name
----+------------+-------------
  1 | ivanov     | ivan
  2 | petrov     | petr
(2 rows)

-- с ноута
psql -p 5432 -U postgres -h 89.169.131.200 -d otus -W
```

### удалить контейнер с сервером
```
yc-user@otus-vm:~$ sudo docker stop 1b323590eaa0
1b323590eaa0
yc-user@otus-vm:~$ sudo docker rm 1b323590eaa0
1b323590eaa0
```

### создать его заново
```
yc-user@otus-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
bb5732de31bb714ec7d645bab1a3fa756d09ad779f57da42a7e7930255c5e238
```

### подключится снова из контейнера с клиентом к контейнеру с сервером
### проверить, что данные остались на месте
```
yc-user@otus-vm:~$  psql -p 5432 -U postgres -h 89.169.131.200 -d otus -W
Password:
psql (15.8 (Ubuntu 15.8-1.pgdg20.04+1))
Type "help" for help.

otus=# select * from students;
 id | first_name | second_name
----+------------+-------------
  1 | ivanov     | ivan
  2 | petrov     | petr
(2 rows)
```


