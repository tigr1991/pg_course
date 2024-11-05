# ДЗ 6

## В Docker замучился делать, было 3 подхода, но что-то не доделывал

## Сделал асинхронную реплику на Digital Ocean

P.s. тоже не всё сразу получалось, поднадоело копипастить IP, поэтому ниже увидите * и 0.0.0.0/0

1. master ```ssh root@164.92.225.10```
2. async реплика ```ssh root@164.92.230.54```
3. ```apt install postgresql``` на обоих
4. На мастере загрузим тайские перевозки
```
mount -o remount,size=2024M /tmp
su postgres
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```
5.  На мастере
```sql
cat >> /etc/postgresql/16/main/postgresql.conf << EOL
listen_addresses = '*' # Не указываю IP для теста, чтоб с IDE подключиться
EOL
```

5. На мастере
```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'qwerty';
SELECT pg_create_physical_replication_slot('async_slot');
```
6. На мастере postgresql.conf
```
cat >> /etc/postgresql/16/main/postgresql.conf << EOL
# Уровень журнала для репликации
wal_level = replica
# Разрешаем несколько одновременных подключений для репликации
max_wal_senders = 3
# Минимальный размер журнала WAL для хранения, специально уменьшаем
wal_keep_size = 8MB
EOL
```
7. На мастере pg_hba.conf (указываю 0.0.0.0/0 только потому, что устал менять IP)
```
cat >> /etc/postgresql/16/main/pg_hba.conf << EOL
host thai replicator 0.0.0.0/0 scram-sha-256
host replication replicator 0.0.0.0/0 scram-sha-256
EOL
```
8. Перезапускаем мастер
```
systemctl restart postgresql
```


9. Останавливаем постгрис на реплике и удаляем папку
```
pg_ctlcluster 16 main stop
rm -rf /var/lib/postgresql/16/main
```

10. Запускаем бэкап
```
pg_basebackup -h 164.92.225.10 -p 5432 -U replicator -R -S async_slot -D /var/lib/postgresql/16/main
```

11. На мастере проверяем реплику:
```sql
postgres=# SELECT
    pid,                                             
    usename AS user,
    application_name,
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM
    pg_stat_replication;


 pid  |    user    | application_name |  client_addr  | state  | sync_state | write_lag | flush_lag | replay_lag 
------+------------+------------------+---------------+--------+------------+-----------+-----------+------------
 4591 | replicator | pg_basebackup    | 164.92.230.54 | backup | async      |           |           | 
(1 row)


 -- На реплике     
postgres=# SELECT
                     pg_last_wal_replay_lsn() AS last_replay_location,
                     pg_is_in_recovery() AS is_in_recovery;
last_replay_location | is_in_recovery 
----------------------+----------------
 0/226FE6A8           | t


-- На мастере
postgres=# SELECT pg_current_wal_lsn() AS master_current_lsn;
master_current_lsn
--------------------
0/226FE6A8
(1 row)

-- На реплике
postgres=# SELECT pg_last_wal_replay_lsn() AS replica_last_replay_lsn;
replica_last_replay_lsn
-------------------------
0/226FE6A8
(1 row)

```

### Текст ДЗ
1. Сделать асинхронную реплику 
2. Сделать синхронную реплику (*)
3. Сделать синхронную реплику + асинхронную каскадно снимаемую с синхронной (*)
