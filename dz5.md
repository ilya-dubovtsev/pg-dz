## ДЗ №5
0. На одном кластере
```
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
```
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```

```
cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
	ceil(random()*100)
	, (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL
```

```
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 50978
number of failed transactions: 0 (0.000%)
latency average = 1.569 ms
initial connection time = 15.973 ms
tps = 5099.673620 (without initial connection time)
```

```
cat > ~/workload.sql << EOL

\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL
```

```
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 228451
number of failed transactions: 0 (0.000%)
latency average = 0.350 ms
initial connection time = 20.707 ms
tps = 22870.570955 (without initial connection time)

```

1. Развернуть асинхронную реплику
```
pg_createcluster 16 main2
Creating new PostgreSQL cluster 16/main2 ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/main2 --auth-local peer --auth-host scram-sha-256 --no-instructions
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных отключён.

исправление прав для существующего каталога /var/lib/postgresql/16/main2... ок
создание подкаталогов... ок
выбирается реализация динамической разделяемой памяти... posix
выбирается значение max_connections по умолчанию... 100
выбирается значение shared_buffers по умолчанию... 128MB
выбирается часовой пояс по умолчанию... Europe/Moscow
создание конфигурационных файлов... ок
выполняется подготовительный скрипт... ок
выполняется заключительная инициализация... ок
сохранение данных на диске... ок
Ver Cluster Port Status Owner    Data directory               Log file
16  main2   5433 down   postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
```
```
cat >> /etc/postgresql/16/main/postgresql.conf << EOL
listen_addresses = 'localhost'
EOL
```
```
psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret$123';"
```
```
psql -c "SELECT pg_create_physical_replication_slot('test');"
```
```
cat >> ~/.pgpass << EOL
localhost:5432:*:replicator:secret$123
EOL
```

```
chmod 0600 ~/.pgpass
```

```
pg_ctlcluster 16 main2 stop
```

```
rm -rf /var/lib/postgresql/16/main2
```
```
psql
checkpoint;
```
```
pg_basebackup -h postgres4 -p 5432 -U replicator -R -S test -D /var/lib/postgresql/16/main
```

```
pg_lsclusters 
Ver Cluster Port Status        Owner    Data directory               Log file
16  main    5432 online        postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 down,recovery postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
```
```
pg_ctlcluster 16 main2 start
```

```
pg_lsclusters 
Ver Cluster Port Status          Owner    Data directory               Log file
16  main    5432 online          postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 online,recovery postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log

```
```
psql -d thai -c "select pg_is_in_recovery();"
 pg_is_in_recovery 
-------------------
 f
(1 строка)

postgres@test1:~$ psql -p 5433 -d thai -c "select pg_is_in_recovery();"
 pg_is_in_recovery 
-------------------
 t
(1 строка)

```
```
psql -p 5433 -d thai -c "select count(*) from book.tickets;"
  count  
---------
 5236483
(1 строка)

```
2. Тестируем производительность по сравнению с сингл инстансом

```
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 30511
number of failed transactions: 0 (0.000%)
latency average = 2.622 ms
initial connection time = 16.587 ms
tps = 3051.111289 (without initial connection time)
```
```
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 226721
number of failed transactions: 0 (0.000%)
latency average = 0.352 ms
initial connection time = 15.374 ms
tps = 22696.101127 (without initial connection time)
```
```
на реплике
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432  thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 227563
number of failed transactions: 0 (0.000%)
latency average = 0.351 ms
initial connection time = 15.398 ms
tps = 22761.521493 (without initial connection time)
```

```
psql
SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 232101
usesysid         | 16799
usename          | replicator
application_name | 16/main2
client_addr      | ::1
client_hostname  | 
client_port      | 37588
backend_start    | 2024-10-23 23:32:45.943691+03
backend_xmin     | 
state            | streaming
sent_lsn         | 1/42A2C380
write_lsn        | 1/42A2C380
flush_lsn        | 1/42A2C380
replay_lsn       | 1/42A2C380
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2024-10-23 23:46:08.164139+03
```

```
ps -o pid,command --ppid `head -n 1 /var/lib/postgresql/16/main/postmaster.pid`
    PID COMMAND
   4462 postgres: 16/main: checkpointer 
   4463 postgres: 16/main: background writer 
   4465 postgres: 16/main: walwriter 
   4466 postgres: 16/main: autovacuum launcher 
   4467 postgres: 16/main: logical replication launcher 
 231772 postgres: 16/main: postgres postgres [local] idle
 232101 postgres: 16/main: walsender replicator ::1(37588) streaming 1/42A2C380
```

```
ps -o pid,command --ppid `head -n 1 /var/lib/postgresql/16/main2/postmaster.pid`
    PID COMMAND
 232097 postgres: 16/main2: checkpointer 
 232098 postgres: 16/main2: background writer 
 232099 postgres: 16/main2: startup recovering 000000010000000100000042
 232100 postgres: 16/main2: walreceiver streaming 1/42A2C380
```

3. Задание со * переделать под синхронную реплику
```
postgres=# show synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 строка)

postgres=# alter system set synchronous_commit = 'remote_apply';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 строка)

postgres=# show synchronous_commit;
 synchronous_commit 
--------------------
 remote_apply
(1 строка)
```

```
в мастере
alter system set synchronous_standby_names = 'master1';
SELECT pg_reload_conf();
```

```
в реплике
nano /var/lib/postgresql/16/main2/postgresql.auto.conf
primary_conninfo = '... application_name=master1'
SELECT pg_reload_conf();
```

```
в мастере
SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 233309
usesysid         | 16799
usename          | replicator
application_name | master1
client_addr      | ::1
client_hostname  | 
client_port      | 45270
backend_start    | 2024-10-24 01:17:51.043653+03
backend_xmin     | 
state            | streaming
sent_lsn         | 1/43E54118
write_lsn        | 1/43E54118
flush_lsn        | 1/43E54118
replay_lsn       | 1/43E54118
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 1
sync_state       | sync
reply_time       | 2024-10-24 01:23:31.218671+03
```

```
делаем измерения
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 20531
number of failed transactions: 0 (0.000%)
latency average = 3.893 ms
initial connection time = 17.861 ms
tps = 2055.114012 (without initial connection time)

/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432  thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 228142
number of failed transactions: 0 (0.000%)
latency average = 0.350 ms
initial connection time = 14.721 ms
tps = 22841.399539 (without initial connection time)

```