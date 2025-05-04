## postgres_fdw

Выполнил следующие команды 
```sql
CREATE EXTENSION postgres_fdw;

CREATE EXTENSION dblink;

CREATE SERVER postgres16_server2
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres16', port '5432', dbname 'thai');

CREATE USER MAPPING FOR postgres
    SERVER postgres16_server2
    OPTIONS (user 'postgres', password 'postgres');

CREATE DATABASE thai;
CREATE SCHEMA IF NOT EXISTS book;

SELECT * FROM dblink(
                      'postgres16_server2',
                      'SELECT nspname FROM pg_namespace WHERE nspname = ''book'''
              ) AS t(name text);

-- Импортируем ТОЛЬКО таблицу thai из схемы book
create schema book;

IMPORT FOREIGN SCHEMA book
    FROM SERVER postgres16_server2 INTO book;
```
Последний запрос выполнился быстро
И после замерил выполнение данного запроса
```sql
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
```
получилось 23 секунды

## PG_dump/PG_restore

Выполнял дапм базы 
```bash
time docker exec postgres16 pg_dump -U postgres -F d -j 4 -f /tmp/dump -n book thai                                                   
```
заняло 1 минуту 36 секунд 
```text
docker exec postgres16 pg_dump -U postgres -F d -j 4 -f /tmp/dump -n book tha  0.01s user 0.01s system 0% cpu 1:03.36 total
```
и команда для рестора
```bash
time docker exec postgres17 pg_restore -U postgres -F d -j 4 -d thai2 /tmp/dump
```
заняло 1 минуту 8 секунд
```text
docker exec postgres17 pg_restore -U postgres -F d -j 4 -d thai2 /tmp/dump  0.01s user 0.02s system 0% cpu 1:08.05 total
```

и время выполнения данного запроса 
```sql
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
```
заняло так же 23 секунды