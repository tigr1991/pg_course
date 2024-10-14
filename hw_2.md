# ДЗ 2

1 +

2 +

3 +

4 +
```
postgres=# SELECT * FROM test1;
id | val
----+-----
1 | aaa
2 | bbb
3 | ccc
(3 rows)
```

5 +

```
postgres=# SHOW transaction_isolation;
 transaction_isolation
-----------------------
read committed
(1 row)
```
6 +

7 +

8 +

9 : НЕ ВИЖУ, т.к. изменения в первой вкладке НЕ закоммичены, а уровень изоляции второй вкладки READ COMMITTED

10 +

11 +

12 : ВИЖУ, т.к. изменения были закоммичены, и у нас уровень изоляции ВСЕГО ЛИШЬ read committed, был бы выше, не увидели бы

13 +

14 + `BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;`

15 +

16 +

17 : НЕ ВИЖУ, т.к. уровень REPEATABLE READ и транзакция была начата до изменений

18 +

19 +

20 : НЕ ВИЖУ, т.к. уровень REPEATABLE READ, гарантировано повторное чтение