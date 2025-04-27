Создадим таблицу и зальем данные 
```sql
drop table if exists t;
CREATE TABLE t AS
SELECT i AS id, (SELECT jsonb_object_agg(j, j) FROM generate_series(1, 1000) j) js
FROM generate_series(1, 10000) i;

SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
```
После вставки данных
```json
{
  "heap_rel": "t",
  "heap_rel_size": "512 kB",
  "toast_rel": "pg_toast.pg_toast_40023",
  "toast_rel_size": "78 MB"
}
```
Создадим индексы
```sql
-- GIN-индекс для всего JSONB-поля (универсальный поиск)
CREATE INDEX idx_t_js_gin ON t USING GIN (js);

-- Индекс по конкретному ключу (например, для часто используемого ключа "1")
CREATE INDEX idx_t_js_key1 ON t ((js->>'1'));

```
Несколько раз обновим данные
```sql
-- Обновляем значение ключа "1" для id = 42
UPDATE t
SET js = jsonb_set(js, '{1}', '9999'::jsonb)
WHERE id = 42;

-- Массовое обновление (изменяем все чётные ключи)
UPDATE t
SET js = jsonb_set(js, '{2}', to_jsonb(id*100))
WHERE id % 2 = 0;
```

После обновления данных
```json
{
  "heap_rel": "t",
  "heap_rel_size": "768 kB",
  "toast_rel": "pg_toast.pg_toast_40023",
  "toast_rel_size": "117 MB"
}
```
Варианты решения bloating

1-й Изменение storage 
После изменения storage параметр на EXTERNAL
```sql
-- Меняем storage параметр на EXTERNAL (меньше сжатие, но быстрее доступ)
ALTER TABLE t ALTER COLUMN js SET STORAGE EXTERNAL;

-- Перестраиваем таблицу
VACUUM FULL t;
```
```json
{
  "heap_rel": "t",
  "heap_rel_size": "512 kB",
  "toast_rel": "pg_toast.pg_toast_40023",
  "toast_rel_size": "78 MB"
}
```
2-й Вариант: Нормализация данных (вынос части JSON в отдельные столбцы)

Проверяем размер индексов до обновления данных
```json
[
  {
    "indexname": "idx_t_js_gin",
    "index_size": "47 MB"
  },
  {
    "indexname": "idx_t_js_key1",
    "index_size": "88 kB"
  }
]
```
После обновления данных
```json
[
  {
    "indexname": "idx_t_js_gin",
    "index_size": "82 MB"
  },
  {
    "indexname": "idx_t_js_key1",
    "index_size": "152 kB"
  }
]
```
Выполняем реиндекс 
```sql
REINDEX INDEX idx_t_js_gin;
REINDEX INDEX idx_t_js_key1;
```
Проверяем снова размер индексов
```json
[
  {
    "indexname": "idx_t_js_gin",
    "index_size": "47 MB"
  },
  {
    "indexname": "idx_t_js_key1",
    "index_size": "88 kB"
  }
]
```

