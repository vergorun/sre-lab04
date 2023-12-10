# sre-lab04

## 1.  Отключение узла  
## 1.1. Описание эксперимента:
Предварительно проверяем статус репликации, убеждаемся, что реплика синхронизирована
```
mcuser@db1:~$ sudo -u postgres psql weather -c 'select status,last_msg_send_time,last_msg_receipt_time,slot_name,sender_host,sender_port from pg_stat_wal_receiver;'
status | last_msg_send_time | last_msg_receipt_time | slot_name | sender_host | sender_port
-----------+-------------------------------+-------------------------------+-----------+-------------+-------------
streaming | 2023-12-10 16:19:35.447349+03 | 2023-12-10 16:19:35.449881+03 | db1 | 10.0.10.3 | 5432
(1 row)
```
```
mcuser@db2:~$ sudo -u postgres psql weather -c 'select client_addr, client_hostname, client_port, state, sync_state, reply_time from pg_stat_replication;'
client_addr | client_hostname | client_port | state | sync_state | reply_time
-------------+-----------------+-------------+-----------+------------+-------------------------------
10.0.10.2 | db1-pvt.sre.lab | 61092 | streaming | sync | 2023-12-10 16:19:35.449378+03
(1 row)
```
На приложение поступают внешние запросы (2 эндпоинта, по 5 rps на каждый), генерирующие запросы к кластеру базы данных. 
Отключаем хост DB-мастера:
```
mcuser@db2:~$ sudo poweroff
Connection to db2-pvt.sre.lab closed by remote host.
Connection to db2-pvt.sre.lab closed.
```
## 1.2. Ожидаемые результаты:
Штатное переключение на реплику, кластер etcd по прежнему имеет кворум, из 3х инстансов все доступны, обявление реплики новым мастером должно пройти без проблем, ожидаем что сервис patroni применит результат работы алгоритма консенсуса raft от кластера etcd.

```
etcdctl member list -w table
+------------------+---------+-------+-----------------------+-----------------------+------------+
|        ID        | STATUS  | NAME  |      PEER ADDRS       |     CLIENT ADDRS      | IS LEARNER |
+------------------+---------+-------+-----------------------+-----------------------+------------+
| 28cd7332cee13aa8 | started | etcd1 | http://10.0.10.4:2380 | http://10.0.10.4:2379 |      false |
| b586ded327f9460d | started |   lb1 | http://10.0.10.6:2380 | http://10.0.10.6:2379 |      false |
| e8cc3f7ff72fe07d | started | etcd2 | http://10.0.10.5:2380 | http://10.0.10.5:2379 |      false |
+------------------+---------+-------+-----------------------+-----------------------+------------+
```
## 1.3. Реальные результаты:
```
Dec 10 16:19:46 db1 patroni[626]: INFO:patroni.__main__:no action. I am (db1), a secondary, and following a leader (db2)
Dec 10 16:19:46 db1 patroni[626]: 2023-12-10 16:19:46,986 INFO: no action. I am (db1), a secondary, and following a leader (db2)
Dec 10 16:19:53 db1 patroni[626]: INFO:patroni.ha:Lock owner: db2; I am db1
Dec 10 16:19:53 db1 patroni[626]: INFO:patroni.__main__:no action. I am (db1), a secondary, and following a leader (db2)
Dec 10 16:19:53 db1 patroni[626]: 2023-12-10 16:19:53,326 INFO: no action. I am (db1), a secondary, and following a leader (db2)
Dec 10 16:19:53 db1 patroni[626]: WARNING:patroni.ha:Request failed to db2: GET http://10.0.10.3:8008/patroni (HTTPConnectionPool(host='10.0.10.3', port=8008): Max retries exceeded with url: /patroni (Caused by ProtocolError('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))))
Dec 10 16:19:53 db1 patroni[626]: 2023-12-10 16:19:53,392 WARNING: Request failed to db2: GET http://10.0.10.3:8008/patroni (HTTPConnectionPool(host='10.0.10.3', port=8008): Max retries exceeded with url: /patroni (Caused by ProtocolError('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))))
Dec 10 16:19:53 db1 patroni[626]: INFO:patroni.watchdog.base:Software Watchdog activated with 25 second timeout, timing slack 15 seconds
Dec 10 16:19:53 db1 patroni[626]: 2023-12-10 16:19:53,445 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
Dec 10 16:19:54 db1 patroni[1807879]: server signaled
Dec 10 16:19:54 db1 patroni[626]: INFO:patroni.__main__:promoted self to leader by acquiring session lock
Dec 10 16:19:54 db1 patroni[626]: 2023-12-10 16:19:54,575 INFO: promoted self to leader by acquiring session lock
Dec 10 16:19:54 db1 patroni[626]: INFO:patroni.ha:Lock owner: db1; I am db1
Dec 10 16:19:54 db1 patroni[1807882]: server promoting
Dec 10 16:19:54 db1 patroni[626]: INFO:patroni.__main__:updated leader lock during promote
Dec 10 16:19:54 db1 patroni[626]: 2023-12-10 16:19:54,580 INFO: Lock owner: db1; I am db1
Dec 10 16:19:54 db1 patroni[626]: 2023-12-10 16:19:54,679 INFO: updated leader lock during promote
Dec 10 16:19:55 db1 patroni[626]: INFO:patroni.ha:Lock owner: db1; I am db1
Dec 10 16:19:55 db1 patroni[626]: INFO:patroni.__main__:no action. I am (db1), the leader with the lock
Dec 10 16:19:55 db1 patroni[626]: 2023-12-10 16:19:55,803 INFO: no action. I am (db1), the leader with the lock
```

![db-failover01](https://github.com/vergorun/sre-lab04/assets/36616396/692aa26f-19db-4dd6-aa59-6c202337c97a)
![db-failover01-load](https://github.com/vergorun/sre-lab04/assets/36616396/b9a7727b-15ca-4a14-8f22-8e268d7c5843)
