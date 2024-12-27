# Vagrant-Ansible стeнд для работы с Postgres: Backup + Репликация

## Описание задачи

1) Настроить hot_standby репликацию с использованием слотов
2) Настроить правильное резервное копирование

## Выполнение

*Лабораторный стенд развёрнут на хосте с Windows + WSL2. На Windows установлены Vagrant и VirtualBox. Ansible установлен на WSL.*

*Все файлы для vagrant располагаются в директории windows (win_directory), у меня это - D:\VBox_Projects\postgres\. Команды для работы с vagrant запускаются из той же директории.*

*Все файлы для ansible располагаются в директории wsl (wsl_directory), у меня это - /home/sof/sof/otus_labs/postgres/. Команды для работы с ansible звпускаются из той же директории.*

### 1. Разворачивание стенда при помощи Vagrant.

Создан Vagrantfile *(win_directory)* для разворачивания лабороторного стенда.

Для доступа к вм из wsl в Vagrantfile прописан дополнительный проброс порта `ssh-for-wsl` для каждой вм.
В нём нужно указать ip-адрес хоста и желаемый порт ssh для подключения для каждой вм:

```
box.vm.network "forwarded_port", # дополнительный проброс порта для доступа к ВМ из WSL
			auto_correct: true,
			guest: 22,
			host: boxconfig[:wsl], # Порт для подключения по ssh, берется из переменной :wsl в каждой вм
			host_ip: "192.168.1.8", # Ip-адрес Windows-хоста
			id: "ssh-for-wsl"
```

### 2. Настройка стенда при помощи Ansible.

В файле staging/hosts.yaml *(wsl_directory)* нужно заполнить переменные для выполнения настройки стенда.
Для подключения к ВМ:
```
    host_ip: 192.168.1.8 # windows host ip
    dir_wsl: /home/sof/otus_labs/postgres/ # directory wsl whith ansible files
    dir_vagrant: /mnt/d/VBox_Projects/postgres/ # directory wsl whith vagrant files
	
	ansible_port: 'указать порт для подключения по ssh' # указывается к каждой вм согласно настройкам в Vagrantfile ":wsl"
```
Также в staging/hosts.yaml *(wsl_directory)* добавлены переменные для дальнейшего использования в конфиг файлах:
```
    replicator_password: 'пароль для репликации'
    master_ip: 'ip мастер-базы'
    slave_ip: 'ip реплики'
    barman_ip: 'ip бэкап-сервера'
    barman_user: 'имя пользователя для бэкапирования'
    barman_user_password: 'пароль для barman_user'
```
В файле staging/hosts.yaml *(wsl_directory)* дополнительно прописан localhost для выполнения команд в wsl.
Это требуется, чтобы скопировать ключ private_key для подключения к ВМ из директории windows в директорию wsl.

PLAY `WSL localhost copy private_key` в `postgres.yml` *(wsl_directory)* копирует private_key на wsl.

### 3. Настройка Репликации

PLAY `Configure Replication Postgres` в `postgres.yml` *(wsl_directory)* производит необходимые настройки.

После выполнения PLAY проверим корректность работы.
Создание тестовой базы данных на node1:
```
root@node1# sudo -u postgres psql
psql (14.15 (Ubuntu 14.15-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE otus_test;
CREATE DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)
```
Проверка списка баз на node2:
```
root@node2# sudo -u postgres psql
psql (14.15 (Ubuntu 14.15-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

```
Новая БД отображается на node2.
Задание выполнено успешно.
Дополнительный способ проверки.
На node1:
```
postgres=# select * from pg_stat_replication;

---------------------------------

 pid  | usesysid |   usename   | application_name |  client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time
------+----------+-------------+------------------+---------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------+-------------------------------
 2799 |    16384 | replication | 14/main          | 192.168.57.12 |                 |       48416 | 2024-12-26 19:19:00.037924+00 |          735 | streaming | 0/30009B0 | 0/30009B0 | 0/30009B0 | 0/30009B0  |           |           |            |             0 | async      | 2024-12-26 19:43:15.187894+00
(1 row)

```
На node:
```
postgres=# select * from pg_stat_wal_receiver;


----------------------------------------
  pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name |  sender_host  | sender_port |                                                                                                                                       conninfo

-------+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------+-----------+---------------+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 20173 | streaming | 0/3000000         |                 1 | 0/30009B0   | 0/30009B0   |            1 | 2024-12-26 19:42:35.371249+00 | 2024-12-26 19:42:35.356175+00 | 0/30009B0      | 2024-12-26 19:40:01.954207+00 |           | 192.168.57.11 |        5432 | user=replication password=******** channel_binding=prefer dbname=replication host=192.168.57.11 port=5432 fallback_application_name=14/main sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
(1 row)

```

### 4. Настройка Backup.

PLAY `Configure Barman Backup` в `postgres.yml` *(wsl_directory)* производит необходимые настройки.

Шаг внесения изменений в конфигурационный файл pg_hba.conf для доступа barman-пользователя к БД пропущен.
Необходимые настрйоки уже внесены в шаблон `pg_hba.conf.j2` *(wsl_directory)* и были применены перенесены на node1 и node2 в предыдущем шаге.

После выполнения PLAY проверим корректность работы.
Для начала создадим тестовую таблицу на node1 с записью:
```
otus=# CREATE TABLE test (id int, name varchar(30));
CREATE TABLE

otus=# INSERT INTO test VALUES ('1', 'alex');
INSERT 0 1

otus=# select * from test;
 id | name
----+------
  1 | alex
(1 row)
```

Проверка работы barman:
```
root@barman:~# barman switch-wal node1
The WAL file 000000010000000000000005 has been closed on server 'node1'

root@barman:~# barman cron
Starting WAL archiving for server node1

root@barman:~# barman check node1
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```
Запуск создания бэкапа:
```
root@barman:~# barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20241208T230814
Backup start at LSN: 0/6000060 (000000010000000000000006, 00000060)
Starting backup copy via pg_basebackup for 20241208T230814
WARNING: pg_basebackup does not copy the PostgreSQL configuration files that reside outside PGDATA. Please manually backup the following files:
        /etc/postgresql/14/main/postgresql.conf
        /etc/postgresql/14/main/pg_hba.conf
        /etc/postgresql/14/main/pg_ident.conf

Copy done (time: 3 seconds)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
        000000010000000000000004 from server node1 has been removed
        000000010000000000000005 from server node1 has been removed
Backup size: 41.8 MiB
Backup end at LSN: 0/8000000 (000000010000000000000007, 00000000)
Backup completed (start time: 2024-12-08 23:08:14.349261, elapsed time: 3 seconds)
Processing xlog segments from streaming for node1
        000000010000000000000006
WARNING: IMPORTANT: this backup is classified as WAITING_FOR_WALS, meaning that Barman has not received yet all the required WAL files for the backup consistency.
This is a common behaviour in concurrent backup scenarios, and Barman automatically set the backup as DONE once all the required WAL files have been archived.
Hint: execute the backup command with '--wait'
```
Теперь на node1 удалим тестовые базы данных:
```
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(5 rows)

postgres=# DROP DATABASE otus;
DROP DATABASE

postgres=# DROP DATABASE otus_test;
DROP DATABASE

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(3 rows)
```
Теперь запустим восстановление из бэкапа на сервере barman:
```
root@barman:~# barman list-backup node1
node1 20241208T232827 - thu Dec  26 20:28:29 2024 - Size: 41.8 MiB - WAL Size: 0 B - WAITING_FOR_WALS
node1 20241208T230814 - thu Dec  26 20:08:17 2024 - Size: 41.8 MiB - WAL Size: 55.7 KiB - WAITING_FOR_WALS

root@barman:~# barman recover node1 20241208T230814 /var/lib/postgresql/14/main/ --remote-ssh-comman "ssh postgres@192.168.57.11"
Processing xlog segments from streaming for node1
        00000001000000000000000A
The authenticity of host '192.168.57.11 (192.168.57.11)' can't be established.
ED25519 key fingerprint is SHA256:8eRZ3SUlbiIR0WUbk94C/DxBt/kBAAMb2FuPbgKW7n0.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20241208T230814
Destination directory: /var/lib/postgresql/14/main/
Remote command: ssh postgres@192.168.57.11
WARNING: IMPORTANT: You have requested a recovery operation for a backup that does not have yet all the WAL files that are required for consistency.
Copying the base backup.
WARNING: IMPORTANT: The backup we have recovered IS NOT VALID. Required WAL files for consistency are missing. Please verify that WAL archiving is working correctly or evaluate using the 'get-wal' option for recovery
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

WARNING
The following configuration files have not been saved during backup, hence they have not been restored.
You need to manually restore them in order to start the recovered PostgreSQL instance:

    postgresql.conf
    pg_hba.conf
    pg_ident.conf

Recovery completed (start time: 2024-12-27 02:20:11.432739, elapsed time: 13 seconds)

Your PostgreSQL server has been successfully prepared for recovery!
```
Затем, после перезапуска на node1 postgresql, проверим, восстановились ли БД:
```
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(5 rows)
```
И доступны ли записи в тестовой таблице:
```
postgres=# \c otus
You are now connected to database "otus" as user "postgres".

otus=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test1 | table | postgres
(2 rows)

otus=# select * from public.test;
 id | name
----+------
  1 | alex
(1 row)
```
Восстановление успешно.