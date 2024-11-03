## ДЗ №7
1. Создать таблицу с продажами.
```
DROP TABLE sales;
CREATE TABLE sales(created_at date, sum numeric);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-11-03', 100);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-02-15', 200);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-04-11', 300);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-07-09', 400);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-09-05', 500);
INSERT INTO public.sales (created_at, sum) VALUES ('2024-12-12', 600);
INSERT INTO public.sales (created_at, sum) VALUES (null, 700);
```
3. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
   - через case
```
CREATE OR REPLACE FUNCTION year_third(d date) RETURNS integer AS $$
DECLARE
    m int;
    third_number int;
BEGIN
    m = EXTRACT(MONTH FROM d);
    CASE
        WHEN m > 0 AND m <= 4 THEN third_number = 1;
        WHEN m >= 5 AND m <= 8 THEN third_number = 2;
        WHEN m >= 9 AND m <= 12 THEN third_number = 3;
        ELSE third_number = 0;
    END CASE;
    RETURN third_number;
END;
$$ LANGUAGE plpgsql
RETURNS NULL ON NULL INPUT;
```
   - используя математическую операцию
```
CREATE OR REPLACE FUNCTION year_third1(d date) RETURNS integer AS
$$
DECLARE
    m int;
    third int;
BEGIN
    m = EXTRACT(MONTH FROM d);
    IF m > 0 AND m <= 12 THEN
        third = m / 4;
        IF m % 4 <> 0 THEN
            third = third + 1;
        END IF;
        RETURN third;
    ELSE
        RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql
RETURNS NULL ON NULL INPUT;


CREATE OR REPLACE FUNCTION year_third2(d date) RETURNS integer AS
$$
DECLARE
    m int;
BEGIN
    m = EXTRACT(MONTH FROM d);
    IF m > 0 AND m <= 12 THEN
        RETURN (m - 1) / 4 + 1;
    ELSE
        RETURN 0;
    END IF;
END;
$$ LANGUAGE plpgsql
RETURNS NULL ON NULL INPUT;
```
   - предусмотреть NULL на входе
```
RETURNS NULL ON NULL INPUT;
```
4. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало

```
SELECT created_at, year_third(created_at) FROM sales;
 created_at | year_third 
------------+------------
 2024-11-03 |          3
 2024-02-15 |          1
 2024-04-11 |          1
 2024-07-09 |          2
 2024-09-05 |          3
 2024-12-12 |          3
            |           
(7 rows)

SELECT created_at, year_third1(created_at) FROM sales;
 created_at | year_third1 
------------+-------------
 2024-11-03 |           3
 2024-02-15 |           1
 2024-04-11 |           1
 2024-07-09 |           2
 2024-09-05 |           3
 2024-12-12 |           3
            |            
(7 rows)

SELECT created_at, year_third2(created_at) FROM sales;
 created_at | year_third2 
------------+-------------
 2024-11-03 |           3
 2024-02-15 |           1
 2024-04-11 |           1
 2024-07-09 |           2
 2024-09-05 |           3
 2024-12-12 |           3
            |            
(7 rows)

```