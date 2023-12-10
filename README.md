# sre-lab04

## 1.  Отключение узла  

*Планово остановить один из узлов кластера, чтобы проверить процедуру переключения ролей (failover). - Анализировать время, необходимое для восстановления и как система выбирает новый Master узел (и есть ли вообще там стратегия выбора?).*

### 1.1. Описание эксперимента:
Предварительно проверяем статус репликации, убеждаемся, что реплика синхронизирована (на момент начала эксперимента роль мастер - у db2)
db1:
```
mcuser@db1:~$ sudo -u postgres psql weather -c 'select status,last_msg_send_time,last_msg_receipt_time,slot_name,sender_host,sender_port from pg_stat_wal_receiver;'
status | last_msg_send_time | last_msg_receipt_time | slot_name | sender_host | sender_port
-----------+-------------------------------+-------------------------------+-----------+-------------+-------------
streaming | 2023-12-10 16:19:35.447349+03 | 2023-12-10 16:19:35.449881+03 | db1 | 10.0.10.3 | 5432
(1 row)
```
db2:
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
### 1.2. Ожидаемые результаты:
Штатное переключение на реплику, кластер etcd по прежнему имеет кворум, из 3х инстансов все доступны, обявление реплики DB db1 новым мастером, вместо отключенного хоста db2, должно пройти без проблем, ожидаем что сервис patroni применит результат работы алгоритма консенсуса raft от кластера etcd.

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
### 1.3. Реальные результаты:

хост db2 отключен в 16:19:50
Patroni переключен в 16:19:54
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
PostgresqlDB переключен в 16:19:54
```
2023-12-10 16:19:52 MSK [777-2]  LOG:  replication terminated by primary server
2023-12-10 16:19:52 MSK [777-3]  DETAIL:  End of WAL reached on timeline 2 at 0/1D0000A0.
2023-12-10 16:19:52 MSK [777-4]  FATAL:  could not send end-of-streaming message to primary: SSL connection has been closed unexpectedly
	no COPY in progress
2023-12-10 16:19:52 MSK [762-8]  LOG:  invalid record length at 0/1D0000A0: wanted 24, got 0
2023-12-10 16:19:52 MSK [1807779-1]  FATAL:  could not connect to the primary server: connection to server at "10.0.10.3", port 5432 failed: server closed the connection unexpectedly
		This probably means the server terminated abnormally
		before or while processing the request.
2023-12-10 16:19:52 MSK [762-9]  LOG:  waiting for WAL to become available at 0/1D0000B8
2023-12-10 16:19:54 MSK [756-8]  LOG:  received SIGHUP, reloading configuration files
2023-12-10 16:19:54 MSK [756-9]  LOG:  parameter "synchronous_standby_names" changed to "*"
2023-12-10 16:19:54 MSK [762-10]  LOG:  received promote request
2023-12-10 16:19:54 MSK [1807881-1]  FATAL:  terminating walreceiver process due to administrator command
2023-12-10 16:19:54 MSK [762-11]  LOG:  waiting for WAL to become available at 0/1D0000B8
2023-12-10 16:19:54 MSK [762-12]  LOG:  redo done at 0/1D000028 system usage: CPU: user: 26.45 s, system: 20.66 s, elapsed: 4238670.73 s
2023-12-10 16:19:54 MSK [762-13]  LOG:  last completed transaction was at log time 2023-11-28 22:13:30.066233+03
2023-12-10 16:19:54 MSK [762-14]  LOG:  selected new timeline ID: 3
2023-12-10 16:19:54 MSK [762-15]  LOG:  archive recovery complete
2023-12-10 16:19:54 MSK [760-145]  LOG:  checkpoint starting: force
2023-12-10 16:19:54 MSK [756-10]  LOG:  database system is ready to accept connections
2023-12-10 16:19:54 MSK [760-146]  LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.001 s, total=0.009 s; sync files=2, longest=0.001 s, average=0.001 s; distance=13186 kB, estimate=13388 kB
2023-12-10 16:34:54 MSK [760-147]  LOG:  checkpoint starting: time
2023-12-10 16:35:00 MSK [760-148]  LOG:  checkpoint complete: wrote 61 buffers (0.2%); 0 WAL file(s) added, 0 removed, 0 recycled; write=6.017 s, sync=0.002 s, total=6.023 s; sync files=12, longest=0.002 s, average=0.001 s; distance=108 kB, estimate=12060 kB
2023-12-10 16:49:54 MSK [760-149]  LOG:  checkpoint starting: time
2023-12-10 16:49:55 MSK [760-150]  LOG:  checkpoint complete: wrote 10 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=1.030 s, sync=0.001 s, total=1.074 s; sync files=1, longest=0.001 s, average=0.001 s; distance=24 kB, estimate=10856 kB
2023-12-10 17:04:54 MSK [760-151]  LOG:  checkpoint starting: time
2023-12-10 17:04:55 MSK [760-152]  LOG:  checkpoint complete: wrote 8 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.804 s, sync=0.002 s, total=0.812 s; sync files=1, longest=0.002 s, average=0.002 s; distance=16270 kB, estimate=16270 kB
```
Данные по оставшимся членам кластера patroni
```
# etcdctl get --prefix /
/service/postgres-cluster/config
...
/service/postgres-cluster/history
[[1,83886080,"no recovery target specified","2023-10-22T14:55:15.378571+03:00","db2"],[2,486539424,"no recovery target specified","2023-12-10T16:19:54.622604+03:00","db1"]]
/service/postgres-cluster/initialize
7287633852799271604
/service/postgres-cluster/leader
db1
/service/postgres-cluster/members/db1
{"conn_url":"postgres://10.0.10.2:5432/postgres","api_url":"http://10.0.10.2:8008/patroni","state":"running","role":"master","version":"3.1.0","xlog_location":520093696,"timeline":3}
/service/postgres-cluster/status
{"optime":520093696}
/service/postgres-cluster/sync
{"leader":"db1","sync_standby":null}
```
На время переключения произошла кратковременная деградация сервиса:
![db-failover01](https://github.com/vergorun/sre-lab04/assets/36616396/692aa26f-19db-4dd6-aa59-6c202337c97a)
![db-failover01-load](https://github.com/vergorun/sre-lab04/assets/36616396/b9a7727b-15ca-4a14-8f22-8e268d7c5843)
![db-failover01-load_result](https://github.com/vergorun/sre-lab04/assets/36616396/98def3bc-bceb-488d-b03a-1f7aff1e5a05)

### 1.4. Анализ результатов:
Во время переключения реплика-мастер DB ошибку вернули 22(6+16) реквеста. При среднем значении около 10rps время недоступности оценочно составило 2,2 секунды.
Как и ожидалось переключение прошо штатно, кластер etcd не притерпевал изменения количества членов/ролей, консенсус был достигнут.
Обратное включение реплики и добавление хоста в кластер DB происходит без влияния, все запросы проходят без ошибок во время синхронизации реплики.

## 2. Имитация частичной потери сети
*Использовать инструменты для имитации потери пакетов или разрыва TCP-соединений между узлами. Цель — проверить, насколько хорошо система справляется с временной недоступностью узлов и как быстро восстанавливается репликация.*

### 2.1. Описание эксперимента:
Предварительно проверяем статус репликации, убеждаемся, что реплика синхронизирована (на момент начала эксперимента роль мастер - у db1)

db1:
```
mcuser@db1:~$ sudo -u postgres psql weather -c 'select client_addr, client_hostname, client_port, state, sync_state, reply_time from pg_stat_replication;'
could not change directory to "/root": Permission denied
 client_addr | client_hostname | client_port |   state   | sync_state |          reply_time
-------------+-----------------+-------------+-----------+------------+-------------------------------
 10.0.10.3   | db2-pvt.sre.lab |       28084 | streaming | sync       | 2023-12-10 18:41:05.054505+03
(1 row)
```

db2:
```
mcuser@db2:~$ sudo -u postgres psql weather -c 'select status,last_msg_send_time,last_msg_receipt_time,slot_name,sender_host,sender_port from pg_stat_wal_receiver;'
could not change directory to "/root/chaosblade-1.7.2": Permission denied
  status   |      last_msg_send_time       |     last_msg_receipt_time     | slot_name | sender_host | sender_port
-----------+-------------------------------+-------------------------------+-----------+-------------+-------------
 streaming | 2023-12-10 18:40:29.866487+03 | 2023-12-10 18:40:29.872148+03 | db2       | 10.0.10.2   |        5432
(1 row)
```
Создаем потери между хостами db1 и db2, процент потерь устанавливаем 50% на стороне db2 при отправке трафика в сторону db1.

db2:
```
mcuser@db2:~$ sudo ./blade create network corrupt --percent 50 --destination-ip 10.0.10.2 --interface ens160
{"code":200,"success":true,"result":"33d9b02b8362c958"}
```

```
mcuser@db2:~$ sudo ./blade status --type create
{
	"code": 200,
	"success": true,
	"result": [
		{
			"Uid": "33d9b02b8362c958",
			"Command": "network",
			"SubCommand": "corrupt",
			"Flag": " --percent=50 --interface=ens160 --destination-ip=10.0.10.2",
			"Status": "Success",
			"Error": "",
			"CreateTime": "2023-12-10T18:38:34.348051113+03:00",
			"UpdateTime": "2023-12-10T18:38:34.362794817+03:00"
		}
	]
}
```

Запускаем профиль нагрузки из п.1 и добавляем профиль с описациями записи (POST /Forecast, пример на основе locust из https://github.com/vergorun/sre-lab03)

Перодически проверяем статус репликации.

### 2.2. Ожидаемые результаты:
Нагрузка в виде операций чтения из мастера не должна деградировать (операции GET для API приложения), однако при записи (POST/PUT), когда реплика должна подтвердить консистентность записи в синхронном режиме, ожидается деградация, т.к. подтверждения будут теряться, что вызывает большое число TCP-ретрансмитов. 
(для диагностики можно использовать bpftrace/tcpretrans.bt: https://github.com/iovisor/bpftrace/blob/master/tools/tcpretrans.bt)
### 2.3. Реальные результаты:  
Скорость POST снизилась до 1.8rps по сравнению с 10rps зафиксированными при проведении нагрузочного тестирования в отсутствии потерь
```
Type     Name                                                                          # reqs      # fails |    Avg     Min     Max    Med |   req/s  failures/s
--------|----------------------------------------------------------------------------|-------|-------------|-------|-------|-------|-------|--------|-----------
...
--------|----------------------------------------------------------------------------|-------|-------------|-------|-------|-------|-------|--------|-----------
         Aggregated                                                                       552     0(0.00%) |    864     106   26566    310 |    1.80        0.00
```
максимальноые время ответа выросло на порядок (26,5 сек)

При это скорость операций чтения (GET) на практике не изменилась
![db-loss01-load](https://github.com/vergorun/sre-lab04/assets/36616396/a00a97da-96e8-4239-880a-9a3ec86c163e)
![db-loss01-get](https://github.com/vergorun/sre-lab04/assets/36616396/57086e1a-b647-4747-9387-f5e00d1fca9a)
с помощью инструмента tcpretrans, действительно, фиксировалось большое число TCP-ретрансмитов в соединении между мастером и репликой
[db-loss01_tcpretrans.txt](https://github.com/vergorun/sre-lab04/files/13628499/db-loss01_tcpretrans.txt)

Репликация при такой (относительно невысокой) интенсивности записи продолжала выполняться без критических ошибок, фейловера так же не произошло.

### 2.4. Анализ результатов:
Как и ожидалось потери между мастером и репликой привели к существенной деградации скорости записи, из-за возникновения большого числа TCP-ретрансмитов вкупе с использованием синхронной репликации. Чтение при этом осталось на практически прежнем уровне, однако, эта особенность данной реализации. В других схемах, когда реплика может использовать приложением для чтения, а запросы к ней будут напрвляться через подобный интерфейс с наличием потерь - операции чтения, очевидно, тоже будут подвержены деградации.

## 3. Высокая нагрузка на CPU или I/O
*Запустить процессы, которые создают высокую нагрузку на CPU или дисковую подсистему одного из узлов кластера, чтобы проверить, как это влияет на  
производительность кластера в целом и на работу Patroni.*

### 3.1. Описание эксперимента:
Обязательно отключить конфиг предыдущего эксперимента
```
blade status --type create
blade destroy
```

Запускаем профиль нагрузки из п.1 

запускаем нагрузку для CPU и IO
db1 (master)
```
blade create cpu fullload --cpu-percent 80
blade create disk burn --read --path /var/lib/postgresql/15/main
```

повторяем для CPU загрузки в 100%
```
blade create cpu fullload --cpu-percent 100
```

### 3.2. Ожидаемые результаты:
Если устновленный предел утилизирует не все свободные ресурсы, то существенной деградации не должно быть, но при ичерпании свободного пула CPU/IO полностью - должно быть существенное падение среднего rps и увеличение времени ответа.

### 3.3. Реальные результаты:

При 80% утилизации CPU и диска целевой rps сохранялся, существенной просадки не случилось ввиду еще неиспользованных ресурсов, средний latency в приделах +/- 200мс 
![db-cpu-io-stress-80-load](https://github.com/vergorun/sre-lab04/assets/36616396/a56554c3-c8e4-4534-bb96-af1668e54cff)
![db-cpu-io-stress-80-stat](https://github.com/vergorun/sre-lab04/assets/36616396/3678608e-67f1-43c7-8aa6-92a90c2a15a8)

Однако при увеличении нагрузки на CPU до 100% наблюдается существенная деградация и тест останавливается при превышении порога по latency (>400 мс)
![db-cpu-io-stress-100-load](https://github.com/vergorun/sre-lab04/assets/36616396/e5857267-bad1-4cf1-af50-657b227c102e)
![db-cpu-io-stress-100-stat](https://github.com/vergorun/sre-lab04/assets/36616396/d4e6a2ba-214e-4467-a208-def72c3fffa7)

### 3.4. Анализ результатов:

При провышении нагрузки на CPU до 100% видно, что ввод/вывод практически полностью вытеснен нагрузкой в пространстве пользователя (%iowait в выводе mpstat нулевой). Можно сделать вывод, что для данного приложения, деградация при линейном росте нагрузки CPU наступает только при исчерпании ресурса, при начилии свободного ресурса CPU и даже высокой нагрузки на IO существенной деградации не наступает.
Так же можно отметить, что %steal околонулевой, что свидельсвует об отсутвии передподписки по ресурсам vCPU на уровне гипервизора (по крайней мере во время проведения тестов), при наличии "шумных соседей" на гипервизоре результат мог бы быть иным.

## 4. Тестирование систем мониторинга и оповещения
*С помощью chaos engineering можно также проверить, насколько эффективны системы мониторинга и оповещения. Например, можно искусственно вызвать отказ, который должен быть зарегистрирован системой мониторинга, и убедиться, что оповещения доставляются вовремя*

Проверка работы алертинга совместно со сценариями:
1. ”Split-brain": Одновременно изолировать несколько узлов от сети и дать им возможность объявить себя новыми мастер-узлами. Проверить, успеет ли Patroni достичь консенсуса и избежать ситуации "split-brain".
2. Долгосрочная изоляция: Оставить узел изолированным от кластера на длительное время, затем восстановить соединение и наблюдать за процессом синхронизации и восстановления реплики.
3. Сбои сервисов зависимостей: Изучить поведение кластера Patroni при сбоях в сопутствующих сервисах, например, etcd (которые используются для хранения состояния кластера), путем имитации его недоступности или некорректной работы.

### 4.1. Описание эксперимента:
1. Обязательно проверяем что конфиги предыдущих экспериментов деактивированы, при необходимости чистим

```
blade status --type create
blade destroy
```
2. проверяем статус репикации, проверяем состояние patroni на db1, db2
```
patronictl -c /etc/patroni/patroni.yml list
```
3. db2 (replica), нарушаем связность с мастером db1 и с 2мя из 3х узлов etcd
```
blade create network corrupt --percent 100 --destination-ip 10.0.10.2,10.0.10.4,10.0.10.5 --interface ens160
```
4. проверяем, что связности дествительно нет
db2:
```
ping -c 4 -t 5 10.0.10.2
ping -c 4 -t 5 10.0.10.4
ping -c 4 -t 5 10.0.10.5
```
5. проверяем статус репикации
db1 (master)
```
sudo -u postgres psql weather -c 'select status,last_msg_send_time,last_msg_receipt_time,slot_name,sender_host,sender_port from pg_stat_wal_receiver;'
```
db2 (replica):
```
sudo -u postgres psql weather -c 'select client_addr, client_hostname, client_port, state, sync_state, reply_time from pg_stat_replication;'
```
6. проверяем состояние patroni на db1, db2
```
patronictl -c /etc/patroni/patroni.yml list
```
7. проверяем значения ключей в etcd
```
etcdctl get --prefix /
```
8. Пробуем полностью изолировать db2 от db1 и etcd
db2:
```
blade create network corrupt --percent 100 --destination-ip 10.0.10.2,10.0.10.4,10.0.10.5,10.0.10.6 --interface ens160
```
проверяем patroni аналогично п.6, общий статус кластера
```
sudo -u postgres psql weather -c 'select pg_is_in_recovery()'
```
9. Возвращаем связность между db2 и остальными компанентами и проверяем состояние репликации


### 4.2. Ожидаемые результаты:  
После шагов 1-7 ожидаемое поведение: репликация нарушена, но реплика не объявила себя мастером, т.к. patroni на db2 продолжает иметь связность с etcd3, который является частью все еще активного кластера, в котором установлен консенсус (между экземплярами etcd, и между etcd и db1 связность не нарушена). 
На шаге 8 после полной изоляции patroni от etcd кластера - ожидаем, что останется последнее состояние на db1 - master, на db2 отставшая реплика. 
### 4.3. Реальные результаты:
2. До нарушения связности
db1:<br>
![split-before-start-db1-patroni](https://github.com/vergorun/sre-lab04/assets/36616396/d527de2a-876c-455e-81fe-3ab0bdf6f980)
db2:<br>
![split-before-start-db2-patroni](https://github.com/vergorun/sre-lab04/assets/36616396/d47a570c-68f9-4597-8181-edfedc121072)
4. Связность нарушена<br>
![split-db2-lost-connectivity-to-db1_etcd1_etcd2](https://github.com/vergorun/sre-lab04/assets/36616396/2e29eed8-276a-4168-b122-189252392a49)
5. Репликация остановлена<br>
![split-replication-lost-db1](https://github.com/vergorun/sre-lab04/assets/36616396/dee0b9c7-0a53-405e-ac67-123c32e28874)
![split-replication-lost-db2](https://github.com/vergorun/sre-lab04/assets/36616396/aac5a193-9162-4487-96f6-b13f824b94cd)
6. Вывод patroni подвержает, что репликация остановлена, но у реплики по-прежнему роль реплики<br>
![split-patront-replication-stop-db1](https://github.com/vergorun/sre-lab04/assets/36616396/e969688f-52fe-443c-a0b2-351732e08cb4)
![split-patroni-replication-stop-db2](https://github.com/vergorun/sre-lab04/assets/36616396/27079504-eeb7-435f-bab0-d54332d755a8)
7. Ключи в etcd<br>
![split-etcd1-keys](https://github.com/vergorun/sre-lab04/assets/36616396/c41a8ac1-0976-431a-b9fc-520d88f9a2bc)
8. После полной потери связности от db2 до db1 и всех экземпляров etcd<br>
Patroni потерял связность с DCS и перешел в бесконечный цикл поиска<br>
![split-db2-connection-lost](https://github.com/vergorun/sre-lab04/assets/36616396/a95ebdf8-3a27-433b-a34e-65a8ca452af2)
db1 остался единственным мастером, связь с репликой полностью потеряна <br>
![split-db1-replica-lost](https://github.com/vergorun/sre-lab04/assets/36616396/90fd999f-db3a-4ead-82c3-2de6bec1961d)
db2 как реплика находится в recovery, но так и не перешла в состояние master<br>
![split-db2-replica-recovery](https://github.com/vergorun/sre-lab04/assets/36616396/cc3ff909-4be8-49c0-8083-c8d8f8c68726)

9. После восстановления связности реплика успешно синхронизровалась<br>
![split-db1-resync](https://github.com/vergorun/sre-lab04/assets/36616396/cc67f9d0-c967-44c8-8b41-ef42d3ca13a8)
![split-db2-resync](https://github.com/vergorun/sre-lab04/assets/36616396/b86680d8-442f-47fc-8e71-62ae6dd4f272)

### 4.4. Анализ результатов:
Нарушение связности между инстансом базы данных и DCS (etcd) привело к ожидаемым последствиям - при наружешении связности межу инстансами DB нарушается и репликация, но роли конролируются консенсусом DCS, в рассмотренном сценарии реплика продолжает существовать в составе кластера пока есть связность хотя бы с одним экземпляром etcd, а при полной потере связности остается изолированной, но не объявляет себя мастером, а ожидает появления связности с DCS для получения информации о назначенной роли. При восстановлении связности с кластером такая реплика автоматически включается обратно в состав кластера.

При потере связности до 1го из 3х экземпляров etcd нарушения работы также не происходит, т.к. для формирования консенсуса достаточно 2х из 3х экземпляров
![split-etcd-alert](https://github.com/vergorun/sre-lab04/assets/36616396/9c076f4d-e975-4efe-8c54-37dcecefb3fb)


