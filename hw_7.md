# ДЗ 7

## 1
1. \+
2. \+
3. Среднее время MAIN_SQL: 12-20 сек
4. Детальнее ниже

   
4.1. Оптимизация запросов в book.tickets
```sql
SELECT count(t.id) as order_place, t.fkride
FROM book.tickets t
group by t.fkride;
```

Индекс:
```sql
CREATE INDEX idx_tickets_fkride_id ON book.tickets (fkride, id);
```

Результаты:

| Что          | Время ДО изменений (сек) | Время ПОСЛЕ изменений (сек) |
|--------------|------------------------|-----------------------------|
| Основной SQL | 12-20                  |   5-8                     | 
| Подзапрос к book.tickets  | 9-16           |   6 ± 1                       |

4.2. Накидаем индексов по всем таблицам на поля, используемые как внешние ключи, всё подряд

```sql
CREATE INDEX idx_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX idx_schedule_id ON book.schedule(id);

CREATE INDEX idx_schedule_fkroute ON book.schedule(fkroute);
CREATE INDEX idx_busroute_id ON book.busroute(id);

CREATE INDEX idx_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);
CREATE INDEX idx_busstation_id ON book.busstation(id);

CREATE INDEX idx_ride_fkbus ON book.ride(fkbus);
CREATE INDEX idx_seat_fkbus ON book.seat(fkbus);

CREATE INDEX idx_ride_id ON book.ride(id);
```

После создания индексов ситуация НЕ поменялась, как было 5-8 сек, так и осталось.

На всякий случай:
```sql
ANALYZE;
```
Не меняется.

4.3 Удалю все созданные индексы и с помощью нейронки сгенерирую все возможно полезные индексы:
```sql
-- Индексы для таблицы ride
CREATE INDEX idx_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX idx_ride_fkbus ON book.ride(fkbus);
CREATE INDEX idx_ride_id ON book.ride(id);
CREATE INDEX idx_ride_startdate ON book.ride(startdate);

-- Индексы для таблицы schedule
CREATE INDEX idx_schedule_id ON book.schedule(id);
CREATE INDEX idx_schedule_fkroute ON book.schedule(fkroute);

-- Индексы для таблицы busroute
CREATE INDEX idx_busroute_id ON book.busroute(id);
CREATE INDEX idx_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);

-- Индексы для таблицы busstation
CREATE INDEX idx_busstation_id ON book.busstation(id);
CREATE INDEX idx_busstation_city_name ON book.busstation(city, name);

-- Индексы для таблицы tickets (CTE order_place)
CREATE INDEX idx_tickets_fkride ON book.tickets(fkride);

-- Индексы для таблицы seat (CTE all_place)
CREATE INDEX idx_seat_fkbus ON book.seat(fkbus);
```
MAIN SQL остался в районе 13 секунд.

4.4 Удаляю все индексы, ещё раз создаю самый первый на tickets и добавляю на ride
```sql
CREATE INDEX idx_tickets_fkride_id ON book.tickets (fkride, id);
CREATE INDEX idx_ride_id_startdate ON book.ride(id, startdate);
```
По времени всё как и после самого первого пункта (индекса на book.tickets)

# Итого: 

 ## прихожу к выводу, что полезен был только первый индекс, созданный в самом начале. По эксплейну тоже не вижу, что можно улучшить

```sql
Limit  (cost=2220038.56..2220038.58 rows=10 width=56)
   Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
   ->  Sort  (cost=2220038.56..2223981.83 rows=1577308 width=56)
         Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
         Sort Key: r.startdate
         ->  HashAggregate  (cost=2131487.08..2185953.50 rows=1577308 width=56)
               Output: r.id, r.startdate, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
               Group Key: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id))
               Planned Partitions: 32
               ->  Hash Join  (cost=1055.45..1960940.65 rows=1577308 width=56)
                     Output: r.id, ((bs.city || ', '::text) || bs.name), (count(t.id)), (count(s_1.id)), r.startdate
                     Inner Unique: true
                     Hash Cond: (r.fkbus = s_1.fkbus)
                     ->  Hash Join  (cost=1050.34..1945399.05 rows=1577308 width=36)
                           Output: r.id, r.startdate, r.fkbus, bs.city, bs.name, (count(t.id))
                           Inner Unique: true
                           Hash Cond: (br.fkbusstationfrom = bs.id)
                           ->  Hash Join  (cost=1049.12..1939502.64 rows=1577308 width=24)
                                 Output: r.id, r.startdate, r.fkbus, br.fkbusstationfrom, (count(t.id))
                                 Inner Unique: true
                                 Hash Cond: (s.fkroute = br.id)
                                 ->  Hash Join  (cost=1046.77..1935067.40 rows=1577308 width=24)
                                       Output: r.id, r.startdate, r.fkbus, s.fkroute, (count(t.id))
                                       Inner Unique: true
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Merge Join  (cost=1001.02..1930869.51 rows=1577308 width=24)
                                             Output: r.id, r.startdate, r.fkschedule, r.fkbus, (count(t.id))
                                             Inner Unique: true
                                             Merge Cond: (r.id = t.fkride)
                                             ->  Index Scan using ride_pkey on book.ride r  (cost=0.43..47127.43 rows=1500000 width=16)
                                                   Output: r.id, r.startdate, r.fkbus, r.fkschedule
                                             ->  Finalize GroupAggregate  (cost=1000.59..1860275.74 rows=1577308 width=12)
                                                   Output: count(t.id), t.fkride
                                                   Group Key: t.fkride
                                                   ->  Gather Merge  (cost=1000.59..1828729.58 rows=3154616 width=12)
                                                         Output: t.fkride, (PARTIAL count(t.id))
                                                         Workers Planned: 2
                                                         ->  Partial GroupAggregate  (cost=0.56..1463608.59 rows=1577308 width=12)
                                                               Output: t.fkride, PARTIAL count(t.id)
                                                               Group Key: t.fkride
                                                               ->  Parallel Index Only Scan using idx_tickets_fkride_id on book.tickets t  (cost=0.56..1335340.77 rows=22498948 width=12)
                                                                     Output: t.fkride, t.id
                                       ->  Hash  (cost=27.00..27.00 rows=1500 width=8)
                                             Output: s.id, s.fkroute
                                             ->  Seq Scan on book.schedule s  (cost=0.00..27.00 rows=1500 width=8)
                                                   Output: s.id, s.fkroute
                                 ->  Hash  (cost=1.60..1.60 rows=60 width=8)
                                       Output: br.id, br.fkbusstationfrom
                                       ->  Seq Scan on book.busroute br  (cost=0.00..1.60 rows=60 width=8)
                                             Output: br.id, br.fkbusstationfrom
                           ->  Hash  (cost=1.10..1.10 rows=10 width=20)
                                 Output: bs.city, bs.name, bs.id
                                 ->  Seq Scan on book.busstation bs  (cost=0.00..1.10 rows=10 width=20)
                                       Output: bs.city, bs.name, bs.id
                     ->  Hash  (cost=5.05..5.05 rows=5 width=12)
                           Output: (count(s_1.id)), s_1.fkbus
                           ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12)
                                 Output: count(s_1.id), s_1.fkbus
                                 Group Key: s_1.fkbus
                                 ->  Seq Scan on book.seat s_1  (cost=0.00..4.00 rows=200 width=8)
                                       Output: s_1.id, s_1.fkbus, s_1.place, s_1.fkseatcategory
 JIT:
   Functions: 48
   Options: Inlining true, Optimization true, Expressions true, Deforming true


```

### Текст ДЗ
1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
   https://github.com/aeuge/postgres16book/tree/main/database
3. Проверить скорость выполнения сложного запроса (приложен в конце файла скриптов)
4. Навесить индексы на внешние ключ
5. Проверить, помогли ли индексы на внешние ключи ускориться

```sql
EXPLAIN VERBOSE WITH all_place AS (
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