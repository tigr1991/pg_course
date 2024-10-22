# ДЗ 4

## 1 
```sql
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts(id integer, amount numeric);
INSERT INTO accounts (id, amount) VALUES (1, 1991), (2, 1994);
```

## 2
| Terminal1                                             | Terminal2                                             |
|-------------------------------------------------------|-------------------------------------------------------|
| ```BEGIN;```                                          |                                                       |
|                                                       | ```BEGIN;```                                          |
| ```SELECT * FROM accounts WHERE id = 1 FOR UPDATE;``` |                                                       |
|                                                       | ```SELECT * FROM accounts WHERE id = 2 FOR UPDATE;``` |
| ```SELECT * FROM accounts WHERE id = 2 FOR UPDATE;``` |                                                       |
|                                                       | ```SELECT * FROM accounts WHERE id = 1 FOR UPDATE;``` |

T2 output:
```txt
ERROR:  deadlock detected
DETAIL:  Process 53 waits for ShareLock on transaction 898; blocked by process 43.
Process 43 waits for ShareLock on transaction 899; blocked by process 53.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,1) in relation "accounts"
```

## 3
Изначально ничего в логи не попало, пришлось повозиться, чтобы внутри Docker включить логирование для PostgreSQL, в результате лог стал собираться:
```txt
> cat postgresql-2024-10-22_190139.log
.........
2024-10-22 19:03:05.649 UTC [62] ERROR:  deadlock detected
2024-10-22 19:03:05.649 UTC [62] DETAIL:  Process 62 waits for ShareLock on transaction 902; blocked by process 44.
	Process 44 waits for ShareLock on transaction 903; blocked by process 62.
	Process 62: SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
	Process 44: SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
2024-10-22 19:03:05.649 UTC [62] HINT:  See server log for query details.
2024-10-22 19:03:05.649 UTC [62] CONTEXT:  while locking tuple (0,1) in relation "accounts"
2024-10-22 19:03:05.649 UTC [62] STATEMENT:  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
.....
```



### Текст ДЗ
1. Создать таблицу accounts(id integer, amount numeric);
2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).
3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.
