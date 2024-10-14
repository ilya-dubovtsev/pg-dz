## ДЗ №3

1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
```
BEGIN;
DROP TABLE IF EXISTS test_strings;
CREATE TABLE test_strings(name varchar(50));
INSERT INTO test_strings SELECT 'string'||s.id FROM generate_series(1,100000) AS s(id);
COMMIT;
```
```
select * from test_strings limit 10;
   name   
----------
 string1
 string2
 string3
 string4
 string5
 string6
 string7
 string8
 string9
 string10
(10 строк)

```
2. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('test_strings'));
 pg_size_pretty 
----------------
 4360 kB
(1 строка)
```
3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
BEGIN;
UPDATE test_strings SET name = name||'a';
UPDATE test_strings SET name = name||'b';
UPDATE test_strings SET name = name||'c';
UPDATE test_strings SET name = name||'d';
UPDATE test_strings SET name = name||'e';
COMMIT;
```
```
select * from test_strings limit 10;
     name      
---------------
 string1abcde
 string2abcde
 string3abcde
 string4abcde
 string5abcde
 string6abcde
 string7abcde
 string8abcde
 string9abcde
 string10abcde
(10 строк)
```
4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил
   автовакуум
```
SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'test_strings';
 n_dead_tup | last_autovacuum 
------------+-----------------
     500000 | 
(1 строка)
```
5. Подождать некоторое время, проверяя, пришел ли автовакуум
```
SELECT n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'test_strings';
 n_dead_tup |       last_autovacuum        
------------+------------------------------
          0 | 2024-10-15 01:11:14.10495+03
(1 строка)

```
6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
BEGIN;
UPDATE test_strings SET name = name||'a';
UPDATE test_strings SET name = name||'b';
UPDATE test_strings SET name = name||'c';
UPDATE test_strings SET name = name||'d';
UPDATE test_strings SET name = name||'e';
COMMIT;
```
7. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('test_strings'));
 pg_size_pretty 
----------------
 30 MB
(1 строка)

```
8. Отключить Автовакуум на конкретной таблице
```
ALTER TABLE test_strings SET (autovacuum_enabled = false);
```
9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
```
BEGIN;
UPDATE test_strings SET name = name||'a';
UPDATE test_strings SET name = name||'b';
UPDATE test_strings SET name = name||'c';
UPDATE test_strings SET name = name||'d';
UPDATE test_strings SET name = name||'e';
UPDATE test_strings SET name = name||'f';
UPDATE test_strings SET name = name||'g';
UPDATE test_strings SET name = name||'h';
UPDATE test_strings SET name = name||'i';
UPDATE test_strings SET name = name||'j';
COMMIT;
```
10. Посмотреть размер файла с таблицей
```
SELECT pg_size_pretty(pg_total_relation_size('test_strings'));
 pg_size_pretty 
----------------
 61 MB
```
11. Объясните полученный результат
> После первых апдейтов автовакуум пометил удаленные строчки и в следующий апдейт информация писалась в старые
> помеченные строки, поэтому таблица увеличивалась  на ~10%. Когда автовакуум выключили, апдейты стали писаться в новые строки,
> поэтому таблица выросла ~ в 2 раза
12. Не забудьте включить автовакуум)

Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.
```
DO $$DECLARE r record;              
BEGIN                             
    FOR I in 1..10 LOOP
         UPDATE test_strings SET name = name||I;
         RAISE NOTICE 'Номер шага: %', I;
    END LOOP;
END$$;
```