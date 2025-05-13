1-е действие по оптимизации 
Это создать индексы 
```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_taxi_trip_payment_type ON taxi_trip(payment_type);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_taxi_trip_tips_fare ON taxi_trip(tips, fare);
```
Далее можно попробовать создать мат view 
```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS taxi_trip_payment_stats AS
SELECT
    payment_type,
    sum(tips::numeric) as total_tips,
    sum(fare::numeric) as total_fare,
    count(*) as trip_count
FROM taxi_trip
GROUP BY payment_type;
```

Время создания около 24 секунд. И последующие запросы 
```sql
SELECT
    payment_type,
    CASE
        WHEN total_fare + total_tips = 0 THEN 0
        ELSE round(total_tips * 100.0 / NULLIF(total_tips + total_fare, 0))
        END as tips_percent,
    trip_count
FROM taxi_trip_payment_stats
ORDER BY trip_count DESC;
```
Выполняются за 150 мс

2-e действие 
Использовать покрывающий индекс
```sql
CREATE INDEX CONCURRENTLY idx_taxi_trip_covering ON taxi_trip(payment_type) INCLUDE (tips, fare);
```
и использовать СТЕ
```sql
WITH aggregated_data AS (
    SELECT
        payment_type,
        SUM(tips::numeric) AS sum_tips,
        SUM(fare::numeric) AS sum_fare,
        COUNT(*) AS cnt
    FROM taxi_trip
    GROUP BY payment_type
)
SELECT
    payment_type,
    CASE
        WHEN sum_tips + sum_fare = 0 THEN 0
        ELSE ROUND((sum_tips * 100.0) / NULLIF(sum_tips + sum_fare, 0))
        END AS tips_percent,
    cnt AS trip_count
FROM aggregated_data
ORDER BY cnt DESC;
```
Время выполнения 5 секунд