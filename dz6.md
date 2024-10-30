## ДЗ №6

1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
   https://github.com/aeuge/postgres16book/tree/main/database
3. Проверить скорость выполнения сложного запроса (приложен в конце файла скриптов)
```
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
 id | depart_date |       busstation       | order_place | all_place 
----+-------------+------------------------+-------------+-----------
  2 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
  3 | 2000-01-01  | Bankgkok, Suvarnabhumi |          34 |        40
  4 | 2000-01-01  | Bankgkok, Eastern      |          33 |        40
  5 | 2000-01-01  | Bankgkok, Eastern      |          36 |        40
  6 | 2000-01-01  | Bankgkok, Eastern      |          32 |        40
  7 | 2000-01-01  | Bankgkok, Chatuchak    |          37 |        40
  8 | 2000-01-01  | Bankgkok, Chatuchak    |          33 |        40
  9 | 2000-01-01  | Bankgkok, Chatuchak    |          37 |        40
 10 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
  1 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
(10 строк)

Время: 1321.469 мс (00:01.321)

```
4. Навесить индексы на внешние ключ
```
CREATE INDEX book_ride_fkschedule_idx ON book.ride(fkschedule); 
CREATE INDEX book_schedule_fkroute_idx ON book.schedule(fkroute);
CREATE INDEX book_busroute_fkbusstationfrom_idx ON book.busroute(fkbusstationfrom);
CREATE INDEX book_tickets_fkride_idx ON book.tickets(fkride);
CREATE INDEX book_seat_fkbus_idx ON book.seat(fkbus);
```
5. Проверить, помогли ли индексы на внешние ключи ускориться
```
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
 id | depart_date |       busstation       | order_place | all_place 
----+-------------+------------------------+-------------+-----------
  2 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
  3 | 2000-01-01  | Bankgkok, Suvarnabhumi |          34 |        40
  4 | 2000-01-01  | Bankgkok, Eastern      |          33 |        40
  5 | 2000-01-01  | Bankgkok, Eastern      |          36 |        40
  6 | 2000-01-01  | Bankgkok, Eastern      |          32 |        40
  7 | 2000-01-01  | Bankgkok, Chatuchak    |          37 |        40
  8 | 2000-01-01  | Bankgkok, Chatuchak    |          33 |        40
  9 | 2000-01-01  | Bankgkok, Chatuchak    |          37 |        40
 10 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
  1 | 2000-01-01  | Bankgkok, Suvarnabhumi |          38 |        40
(10 строк)

Время: 1392.081 мс (00:01.392)
```

>индексы не помогли, потому что не используются