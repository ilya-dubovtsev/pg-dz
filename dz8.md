## ДЗ №8

1. Сгенерировать таблицу с 1 млн JSONB документов
```
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 1/29866D30
(1 строка)

demo=# CREATE TABLE t AS
SELECT i AS id, (SELECT jsonb_object_agg(j, j) FROM generate_series(1, 10) j) js  
FROM generate_series(1, 1000000) i;
SELECT 1000000
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 1/37E0DD88
(1 строка)

demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('1/29866D30', '1/37E0DD88')) AS wal_size;
 wal_size 
----------
 -230 MB
(1 строка)

```
2. Создать индекс
```
demo=# CREATE INDEX t_js_idx ON bookings.t USING gin (js);
CREATE INDEX
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 1/3940BCB0
(1 строка)

demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('1/37E0DD88', '1/3940BCB0')) AS wal_size;
 wal_size 
----------
 -22 MB
(1 строка)

```
3. Обновить 1 из полей в json
```
SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/EDB48B78

UPDATE t SET js = js - '1';

SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 1/28C79898
(1 строка)

SELECT pg_size_pretty(pg_wal_lsn_diff('1/28C79898', '0/EDB48B78')) AS wal_size;
 wal_size 
----------
 945 MB
(1 строка)

```
4. Убедиться в блоатинге TOAST
```
\timing 
Секундомер включён.
demo=# UPDATE t SET js = js - '1';
UPDATE 1000000
Время: 120987.713 мс (02:00.988)

SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 1/75107878
(1 строка)

SELECT pg_size_pretty(pg_wal_lsn_diff('1/3940BCB0', '1/75107878')) AS wal_size;
 wal_size 
----------
 -957 MB
(1 строка)
```
5. Придумать методы избавится от него и проверить на практике
```
SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/10F41AC8
(1 строка)

Время: 0.897 мс
demo=# VACUUM FUll t;
VACUUM
Время: 51548.047 мс (00:51.548)
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/1E09E118
(1 строка)

Время: 1.108 мс
demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('2/10F41AC8', '2/1E09E118')) AS wal_size;
 wal_size 
----------
 -209 MB
(1 строка)

SELECT c1.oid, c1.reltoastrelid, c2.relname
   FROM pg_class AS c1
   LEFT JOIN pg_class AS c2
          ON c1.reltoastrelid = c2.oid
   WHERE c1.relname = 't';
  oid   | reltoastrelid |     relname     
--------+---------------+-----------------
 261743 |        261746 | pg_toast_261743
(1 строка)

CLUSTER pg_toast.pg_toast_261743 USING pg_toast_261743_index;
CLUSTER
Время: 7.444 мс
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/1E2393C8
(1 строка)

Время: 0.456 мс
demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('2/1E09E118', '2/1E2393C8')) AS wal_size;
 wal_size 
----------
 -1645 kB
(1 строка)

Время: 0.375 мс
demo=# UPDATE t SET js = js - '2';
UPDATE 1000000
Время: 107651.673 мс (01:47.652)

ALTER TABLE t ALTER COLUMN js SET STORAGE main;
ALTER TABLE
Время: 2.743 мс
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/55D63F10
(1 строка)

Время: 1.606 мс
demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('2/1E2393C8', '2/55D63F10')) AS wal_size;
 wal_size 
----------
 -891 MB
(1 строка)

Время: 0.469 мс
demo=# UPDATE t SET js = js - '3';
UPDATE 1000000
Время: 100452.650 мс (01:40.453)
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/890138E8
(1 строка)

Время: 0.856 мс
demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('2/55D63F10', '2/890138E8')) AS wal_size;
 wal_size 
----------
 -819 MB
(1 строка)

Время: 0.385 мс

ALTER TABLE t SET (toast_tuple_target = 4080);
ALTER TABLE
Время: 2.595 мс
demo=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 2/89B99620
(1 строка)

Время: 0.591 мс
demo=# SELECT pg_size_pretty(pg_wal_lsn_diff('2/890138E8', '2/89B99620')) AS wal_size;
 wal_size 
----------
 -12 MB
(1 строка)

Время: 0.521 мс
demo=# UPDATE t SET js = js - '4';
UPDATE 1000000
Время: 85219.055 мс (01:25.219)

```