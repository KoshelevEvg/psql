Создадим таблицу продаж и заполним данные
```sql
CREATE TABLE sales
(
    id           SERIAL PRIMARY KEY,
    sale_date    DATE           NOT NULL,
    amount       DECIMAL(10, 2) NOT NULL,
    product_name VARCHAR(100),
    region       VARCHAR(50)
);


INSERT INTO sales (sale_date, amount, product_name, region)
VALUES ('2023-01-15', 100.50, 'Product A', 'North'),
       ('2023-02-20', 200.75, 'Product B', 'South'),
       ('2023-05-10', 150.00, 'Product A', 'East'),
       ('2023-07-22', 300.25, 'Product C', 'West'),
       ('2023-09-05', 250.50, 'Product B', 'North'),
       ('2023-11-18', 180.75, 'Product D', 'South'),
       ('2023-12-01', 400.00, 'Product A', 'East'),
       ('2023-12-12', 120.30, 'Product C', 'West'),
       ('2023-04-02', 90.45, 'Product B', 'North');

```


Функция-запрос на получение данных о продажах за определенный квартал 
```sql
CREATE OR REPLACE FUNCTION get_all_field(a integer)
    RETURNS Table
            (
                id           INTEGER,
                sale_date    DATE,
                amount       DECIMAL(10, 2),
                product_name VARCHAR(100),
                region       VARCHAR(50)
            )
AS
$$
    BEGIN RETURN QUERY
SELECT s.id, s.sale_date, s.amount, s.product_name, s.region
from sales s
WHERE CASE a
          WHEN 1 THEN EXTRACT(MONTH FROM s.sale_date) BETWEEN 1 AND 4
          WHEN 2 THEN EXTRACT(MONTH FROM s.sale_date) BETWEEN 5 AND 8
          WHEN 3 THEN EXTRACT(MONTH FROM s.sale_date) BETWEEN 9 AND 12
          END;

END;
$$
    LANGUAGE plpgsql
    IMMUTABLE
    RETURNS NULL ON NULL INPUT;

SELECT *
FROM get_all_field(1);
SELECT *
FROM get_all_field(2);
SELECT *
FROM get_all_field(3);
```

Арифметическое использование без Case
```sql
CREATE OR REPLACE FUNCTION get_all_field_math(a integer)
    RETURNS Table
            (
                id           INTEGER,
                sale_date    DATE,
                amount       DECIMAL(10, 2),
                product_name VARCHAR(100),
                region       VARCHAR(50)
            )
AS
$$
BEGIN RETURN QUERY
    SELECT s.id, s.sale_date, s.amount, s.product_name, s.region
    from sales s
    WHERE a BETWEEN 1 AND 3 AND
    CEIL(EXTRACT(MONTH FROM s.sale_date)/4.0) = a;
END;
$$
    LANGUAGE plpgsql
    IMMUTABLE
    RETURNS NULL ON NULL INPUT;
```

Вариант 2
```sql
CREATE OR REPLACE FUNCTION get_all_field_math2(a integer)
    RETURNS TABLE (
        id INTEGER,
        sale_date DATE,
        amount DECIMAL(10, 2),
        product_name VARCHAR(100),
        region VARCHAR(50)
    )
AS
$$
BEGIN
    RETURN QUERY 
    SELECT s.id, s.sale_date, s.amount, s.product_name, s.region
    FROM sales s
    WHERE a BETWEEN 1 AND 3 AND 
          EXTRACT(MONTH FROM s.sale_date) BETWEEN (a-1)*4+1 AND a*4;
END;
$$
LANGUAGE plpgsql
STABLE
RETURNS NULL ON NULL INPUT;
```