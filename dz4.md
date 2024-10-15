## ДЗ №4

1. Создать таблицу accounts(id integer, amount numeric);
```
CREATE TABLE accounts(id integer, amount numeric);
```
2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).
```
BEGIN;
INSERT INTO accounts VALUES (1, 100);
INSERT INTO accounts VALUES (2, 200);
INSERT INTO accounts VALUES (3, 300);
INSERT INTO accounts VALUES (4, 400);
COMMIT;
```
```
в первом терминале
BEGIN;
UPDATE accounts SET amount = 200 WHERE id = 3;
UPDATE accounts SET amount = 500 WHERE id = 4;

во втором
BEGIN;
UPDATE accounts SET amount = 300 WHERE id = 4;
UPDATE accounts SET amount = 400 WHERE id = 3;

ERROR:  deadlock detected
ПОДРОБНОСТИ:  Process 116458 waits for ShareLock on transaction 10957; blocked by process 116329.
Process 116329 waits for ShareLock on transaction 10958; blocked by process 116458.
ПОДСКАЗКА:  See server log for query details.
КОНТЕКСТ:  while updating tuple (0,3) in relation "accounts"

```
3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.

```
SELECT * FROM pg_stat_activity \gx

-[ RECORD 3 ]----+-----------------------------------------------
datid            | 5
datname          | postgres
pid              | 116329
leader_pid       | 
usesysid         | 10
usename          | postgres
application_name | psql
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2024-10-15 16:36:02.086528+03
xact_start       | 2024-10-15 22:23:34.155161+03
query_start      | 2024-10-15 22:23:49.296562+03
state_change     | 2024-10-15 22:24:05.247293+03
wait_event_type  | Client
wait_event       | ClientRead
state            | idle in transaction
backend_xid      | 10961
backend_xmin     | 
query_id         | 
query            | UPDATE accounts SET amount = 500 WHERE id = 4;
backend_type     | client backend
-[ RECORD 4 ]----+-----------------------------------------------
datid            | 5
datname          | postgres
pid              | 116458
leader_pid       | 
usesysid         | 10
usename          | postgres
application_name | psql
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2024-10-15 16:48:52.129008+03
xact_start       | 
query_start      | 2024-10-15 22:24:04.246375+03
state_change     | 2024-10-15 22:24:05.247097+03
wait_event_type  | Client
wait_event       | ClientRead
state            | idle in transaction (aborted)
backend_xid      | 
backend_xmin     | 
query_id         | 
query            | UPDATE accounts SET amount = 400 WHERE id = 3;
backend_type     | client backend

SELECT * FROM accounts;              
 id | amount 
----+--------
  1 |    100
  2 |    200
  3 |    200
  4 |    500
(4 строки)
```