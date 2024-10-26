# ДЗ 8

## 1
```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    name VARCHAR NOT NULL,
    sale_date DATE
);
```

```sql
INSERT INTO sales (name, sale_date)
VALUES 
    ('a1', '2023-01-15'),
    ('a2', '2023-02-10'),
    ('a3', '2023-03-25'),
    ('a4', '2023-04-05'),
    ('a5', '2023-05-20'),
    ('a6', '2023-06-17'),
    ('a7', '2023-07-08'),
    ('a8', '2023-08-13'),
    ('a9', '2023-09-22'),
    ('a10', '2023-10-30'),
    ('a11', '2023-11-14'),
    ('a12', '2023-12-25'),
    ('a0', null);
```

## 2

### a
```sql
CREATE OR REPLACE FUNCTION get_year_part_case(sale_date DATE)
RETURNS INTEGER AS $$
DECLARE
    month INTEGER;
BEGIN
    IF sale_date IS NULL THEN
        RETURN -1;
    END IF;
    month := EXTRACT(MONTH FROM sale_date);
    RETURN CASE 
        WHEN month IS NULL THEN NULL
        WHEN month BETWEEN 1 AND 4 THEN 1
        WHEN month BETWEEN 5 AND 8 THEN 2
        WHEN month BETWEEN 9 AND 12 THEN 3
        ELSE -1 -- не NULL для наглядности
    END;
END;
$$ LANGUAGE plpgsql;
```

### b
```sql
CREATE OR REPLACE FUNCTION get_year_part_math(sale_date DATE)
RETURNS INTEGER AS $$
DECLARE
    month INTEGER;
BEGIN
    IF sale_date IS NULL THEN
        RETURN -1; -- не NULL для наглядности
    END IF;
    
    month := EXTRACT(MONTH FROM sale_date);
    
    RETURN (month + 3) / 4;
END;
$$ LANGUAGE plpgsql;
```

## 3
```sql
SELECT 
    id,
    sale_date,
    get_year_part_case(sale_date) AS year_part_case,
    get_year_part_math(sale_date) AS year_part_math
FROM sales;


id | sale_date  | year_part_case | year_part_math
----+------------+----------------+----------------
  1 | 2023-01-15 |              1 |              1
  2 | 2023-02-10 |              1 |              1
  3 | 2023-03-25 |              1 |              1
  4 | 2023-04-05 |              1 |              1
  5 | 2023-05-20 |              2 |              2
  6 | 2023-06-17 |              2 |              2
  7 | 2023-07-08 |              2 |              2
  8 | 2023-08-13 |              2 |              2
  9 | 2023-09-22 |              3 |              3
 10 | 2023-10-30 |              3 |              3
 11 | 2023-11-14 |              3 |              3
 12 | 2023-12-25 |              3 |              3
 13 |            |             -1 |             -1
(13 rows)
```


### Текст ДЗ
ДЗ
1. Создать таблицу с продажами.
2. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
   
   а. через case
      
   b. * (бонуса в виде зачета дз не будет) используя математическую операцию (лучше 2+ варианта)
         
   с. предусмотреть NULL на входе
3. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало