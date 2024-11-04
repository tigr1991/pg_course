# ДЗ 8

## 1 Таблица с данными

### 1 Пересоздаю табличку
```sql
DROP TABLE IF EXISTS test_jsonb;

CREATE TABLE test_jsonb (
   id SERIAL PRIMARY KEY,
   data JSONB
);
```

### 2 Использую функцию из лекции
    
```sql
CREATE OR REPLACE FUNCTION generate_random_string(
  length INTEGER,
  characters TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT AS
$$
DECLARE
result TEXT := '';
BEGIN
  IF length < 1 then
      RAISE EXCEPTION 'Invalid length';
END IF;
FOR __ IN 1..length LOOP
    result := result || substr(characters, floor(random() * length(characters))::int + 1, 1);
end loop;
RETURN result;
END;
$$ LANGUAGE plpgsql;
```

### 3 Для попадания данных в toast, а не в саму таблицу, использую длинные строки, не стал делать 1млн записей, слишком долго

```sql
INSERT INTO test_jsonb (data)
SELECT jsonb_build_object('text', (generate_random_string(1024 * 10)))
FROM generate_series(1, 10000);
```

### 4 Проверим где лежат данные
```sql
SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 'test_jsonb';
```

```
  heap_rel  | heap_rel_size |        toast_rel         | toast_rel_size 
------------+---------------+--------------------------+----------------
 test_jsonb | 512 kB        | pg_toast.pg_toast_592697 | 117 MB
```


## 2 Индекс
### 1 Создаём
```sql
CREATE INDEX idx_test_jsonb_data ON test_jsonb USING GIN (data);
```

### 2 Проверим размер
```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_test_jsonb_data')) AS index_size;
```
```
 index_size 
------------
 736 kB
```

## 3 Обновим поле
```sql
UPDATE test_jsonb
SET data = jsonb_set(data, '{text}', to_jsonb((data->>'text') || '0'), true);
```

## 4 Проверка размера 
```sql
  heap_rel  | heap_rel_size |        toast_rel         | toast_rel_size 
------------+---------------+--------------------------+----------------
 test_jsonb | 1024 kB       | pg_toast.pg_toast_592697 | 234 MB
            
            
 
  index_size 
------------
 1208 kB
```

## 5 Убираем блоатинг

```sql
VACUUM FULL;
```
```
  heap_rel  | heap_rel_size |        toast_rel         | toast_rel_size 
------------+---------------+--------------------------+----------------
 test_jsonb | 512 kB        | pg_toast.pg_toast_592697 | 117 MB
 
 
  index_size 
------------
 736 kB
```

## 6 Блоатинг индекса тоже ушёл

**Внимание!** VACUUM FULL блокирует таблицу! Но для нашего случая это было допустимо.

### Текст ДЗ
1. Сгенерировать таблицу с 1 млн JSONB документов
2. Создать индекс
3. Обновить 1 из полей в json
4. Убедиться в блоатинге TOAST
5. Придумать методы избавится от него и проверить на практике
6. Не забываем про блоатинг индексов*