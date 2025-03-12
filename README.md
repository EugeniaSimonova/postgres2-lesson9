# postgres 2-lesson 9

Имеется 2 кластера:
````
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
16  main    5432 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main1   5433 down   postgres /var/lib/postgresql/16/main1 /var/log/postgresql/postgresql-16-main1.log
````


1. Устанавливаем pg_probackup:

````
$ sudo apt install pg-probackup-16

````

2. Создаем каталог для бекапов:

````
$ sudo rm -rf /home/backups && sudo mkdir /home/backups && sudo chmod 777 /home/backups

````
3. Инициализируем каталог для бекапов:

````
$ sudo su postgres
$ pg_probackup-16 init -B /home/backups

INFO: Backup catalog '/home/backups' successfully initialized

````

4. Инициализируем инстанс кластера, назовем его 'main' и определим, что он будет хранить бэкапы по выбранному пути:

````
$ pg_probackup-16 add-instance --instance 'main' -D /var/lib/postgresql/16/main -B /home/backups

INFO: Instance 'main' successfully initialized
````

5. Создаём таблицу:

````
$ psql

CREATE TABLE books (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  public_year SMALLINT NULL);

INSERT INTO books(title, public_year) VALUES
('Mrs. Dalloway', 1925),
('To the Lighthouse', 1927),
('To Kill a Mockingbird', 1960),
('The Great Gatsby', 1925);
INSERT 0 4
````

6. Посмотрим текущие настройки `pg_probackup`:

````
$ pg_probackup-16 show-config --instance main -B /home/backups
# Backup instance information
pgdata = /var/lib/postgresql/16/main
system-identifier = 7480920599293392840
xlog-seg-size = 16777216
# Connection parameters
pgdatabase = postgres
# Replica parameters
replica-timeout = 5min
# Archive parameters
archive-timeout = 5min
# Logging parameters
log-level-console = INFO
log-level-file = OFF
log-format-console = PLAIN
log-format-file = PLAIN
log-filename = pg_probackup.log
log-rotation-size = 0TB
log-rotation-age = 0d
# Retention parameters
retention-redundancy = 0
retention-window = 0
wal-depth = 0
# Compression parameters
compress-algorithm = none
compress-level = 1
# Remote access parameters
remote-proto = ssh
````

7. Делаем полный бэкап:

````
$ pg_probackup-16 backup --instance 'main' -b FULL --stream --temp-slot -B /home/backups

````

8. Просматриваем имеющиеся бэкапы:

````
$ pg_probackup-16 show -B /home/backups

BACKUP INSTANCE 'main'
================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time  Data   WAL  Zratio  Start LSN  Stop LSN   Status 
================================================================================================================================
 main      16       ST0KXG  2025-03-12 14:15:17+00  FULL  STREAM    1/0   13s  22MB  16MB    1.00  0/2000028  0/20001A0  OK

````

9. Добавляем данные в БД:

````
$ psql

postgres=# INSERT INTO books(title, public_year) VALUES ('The Lord of the Rings',1955);
INSERT 0 1

````

10 Cоздаем инкрементальную копию:
```` 
pg_probackup-16 backup --instance 'main' -b DELTA --stream --temp-slot -B /home/backups

$ pg_probackup-16 show -B /home/backups

BACKUP INSTANCE 'main'
==================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time   Data   WAL  Zratio  Start LSN  Stop LSN   Status 
==================================================================================================================================
 main      16       ST0L5Q  2025-03-12 14:20:16+00  DELTA  STREAM    1/1   10s  138kB  32MB    1.00  0/4000028  0/4000168  OK     
 main      16       ST0KXG  2025-03-12 14:15:17+00  FULL   STREAM    1/0   13s   22MB  16MB    1.00  0/2000028  0/20001A0  OK     

````

11. Удаляем содержимое папки с данными второго кластера:

````
$ rm -rf /var/lib/postgresql/16/main1/*
````

12. Восстанавливаем бэкап во второй кластер:
````
$ pg_probackup-16 restore --instance 'main' -i 'ST0KXG' -D  /var/lib/postgresql/16/main1 -B /home/backups

````

13. Запускаем второй кластер: 

````
$ pg_ctlcluster 16 main1 start
````

14.Проверяем, что во втором кластере присутствует таблица из первого:

````
$ psql -p 5433

postgres=# SELECT * from books;
 id |         title         | public_year 
----+-----------------------+-------------
  1 | Mrs. Dalloway         |        1925
  2 | To the Lighthouse     |        1927
  3 | To Kill a Mockingbird |        1960
  4 | The Great Gatsby      |        1925
(4 rows)

````

15. Останавливаем кластер, удаляем данные и восстанавливаем инкрементальную копию:

````
$ pg_ctlcluster 16 main1 stop
$ rm -rf /var/lib/postgresql/16/main1/*
$ pg_probackup-16 restore --instance 'main' -i 'ST0L5Q' -D /var/lib/postgresql/16/main1 -B /home/backups
$ pg_ctlcluster 16 main1 start

````

16. Проверяем:

````
$ psql -p 5433
postgres=# SELECT * from books;
 id |         title         | public_year 
----+-----------------------+-------------
  1 | Mrs. Dalloway         |        1925
  2 | To the Lighthouse     |        1927
  3 | To Kill a Mockingbird |        1960
  4 | The Great Gatsby      |        1925
  5 | The Lord of the Rings |        1955
(5 rows)

````
В выводе присутствует 5 строка, которая была только в инкрементальной копии.
