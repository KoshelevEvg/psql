Время выполнения изначального запроса 

```postgresql
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
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as
                            busstation,
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
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name,
         t.order_place,st.all_place
ORDER BY r.startdate
limit 10;

Execution Time: 387.818 ms
```

Query plan
```postgresql
Limit  (cost=209172.41..209172.43 rows=10 width=56) (actual time=367.962..368.026 rows=10 loops=1)
  ->  Sort  (cost=209172.41..209532.53 rows=144047 width=56) (actual time=346.319..346.382 rows=10 loops=1)
        Sort Key: r.startdate
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Group  (cost=203538.78..206059.60 rows=144047 width=56) (actual time=324.863..339.864 rows=144000 loops=1)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Sort  (cost=203538.78..203898.90 rows=144047 width=56) (actual time=324.842..329.407 rows=144000 loops=1)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Sort Method: external merge  Disk: 7864kB
                    ->  Hash Join  (cost=1052.96..186272.21 rows=144047 width=56) (actual time=21.807..310.655 rows=144000 loops=1)
                          Hash Cond: (r.fkbus = s_1.fkbus)
                          ->  Hash Join  (cost=1047.85..184848.23 rows=144047 width=36) (actual time=21.752..295.155 rows=144000 loops=1)
                                Hash Cond: (br.fkbusstationfrom = bs.id)
                                ->  Hash Join  (cost=1046.63..184308.63 rows=144047 width=24) (actual time=21.740..284.108 rows=144000 loops=1)
                                      Hash Cond: (s.fkroute = br.id)
                                      ->  Hash Join  (cost=1044.28..183901.45 rows=144047 width=24) (actual time=21.727..272.682 rows=144000 loops=1)
                                            Hash Cond: (r.fkschedule = s.id)
                                            ->  Merge Join  (cost=1000.88..183478.81 rows=144047 width=24) (actual time=21.469..259.837 rows=144000 loops=1)
                                                  Merge Cond: (r.id = t.fkride)
                                                  ->  Index Scan using ride_pkey on ride r  (cost=0.42..4534.42 rows=144000 width=16) (actual time=0.068..14.402 rows=144000 loops=1)
                                                  ->  Finalize GroupAggregate  (cost=1000.46..176783.80 rows=144047 width=12) (actual time=21.390..230.260 rows=144000 loops=1)
                                                        Group Key: t.fkride
                                                        ->  Gather Merge  (cost=1000.46..173902.86 rows=288094 width=12) (actual time=21.349..217.550 rows=162617 loops=1)
                                                              Workers Planned: 2
                                                              Workers Launched: 2
                                                              ->  Partial GroupAggregate  (cost=0.43..139649.64 rows=144047 width=12) (actual time=1.430..203.678 rows=54206 loops=3)
                                                                    Group Key: t.fkride
                                                                    ->  Parallel Index Only Scan using idx_tickets_fkride_id on tickets t  (cost=0.43..127406.07 rows=2160620 width=12) (actual time=0.240..154.070 rows=1728502 loops=3)
                                                                          Heap Fetches: 0
                                            ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.251..0.251 rows=1440 loops=1)
                                                  Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                  ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.047..0.195 rows=1440 loops=1)
                                      ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.008..0.009 rows=60 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                            ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.003..0.005 rows=60 loops=1)
                                ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=0.006..0.006 rows=10 loops=1)
                                      Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                      ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=0.004..0.004 rows=10 loops=1)
                          ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.042..0.043 rows=5 loops=1)
                                Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.040..0.040 rows=5 loops=1)
                                      Group Key: s_1.fkbus
                                      Batches: 1  Memory Usage: 24kB
                                      ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.014..0.018 rows=200 loops=1)
Planning Time: 2.002 ms
JIT:
  Functions: 57
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 7.651 ms (Deform 0.637 ms), Inlining 0.000 ms, Optimization 6.634 ms, Emission 18.608 ms, Total 32.893 ms"
Execution Time: 378.205 ms

```

После добавления следующих индексов

Основные индексы для JOIN операций:
```postgresql
CREATE INDEX IF NOT EXISTS idx_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX IF NOT EXISTS idx_schedule_fkroute ON book.schedule(fkroute);
```

Специальный индекс для агрегации по билетам:
```postgresql
CREATE INDEX IF NOT EXISTS idx_tickets_fkride_covering ON book.tickets(fkride) INCLUDE (id);
```
Индекс для сортировки:
```postgresql
CREATE INDEX IF NOT EXISTS idx_ride_startdate ON book.ride(startdate);
```

После добавления индексов скорость отработки не изменилась
Основываясь на полученном плане после добавления индексов, он остался без изменений

### Предположения из-за чего это произошло

#### Незначительный размер маленьких таблиц
- Таблицы с малым объемом данных:
    - `seat` (200 строк)
    - `busroute` (60 строк)
    - `busstation` (10 строк)
- PostgreSQL предпочитает последовательное сканирование (Seq Scan) для небольших таблиц
- Индексы на маленьких таблицах часто не используются оптимизатором

#### Уже оптимальный доступ к таблице tickets
- Запрос уже использует эффективный доступ:  
  `Parallel Index Only Scan using idx_tickets_fkride_id`
- Дальнейшая оптимизация доступа к этой таблице через индексы невозможна
- Таблица tickets уже сканируется оптимальным способом
