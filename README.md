# pgcloud-homework01

## Домашнее задание
### Работа с уровнями изоляции транзакции в PostgreSQL

**Цель:**
Научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

**Описание/Пошаговая инструкция выполнения домашнего задания:**

В домашней работе использую собственную ВМ Ubuntu и установленный docker в ней.

Установим и запустим контейнер с postgresql
```
# docker run --name jf-postgres -p 5432:5432 -e POSTGRES_PASSWORD=postgres -d postgres
```
Подключимся с 2х терминалов к нашему серверу
```
$ psql -U postgres -h localhost -p 5432
Password for user postgres:
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```
Выключаем автокоммит в обоих сеансах
```
postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```
В первой сессии
```
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```
Проверим текущий уровень изоляции
```
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
**read committed**
Начнем новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции
 В первой сессии добавим новую запись
```
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
Сделаем **select * from persons** во второй сессии
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Мы не еще вставленную строку ('sergey', 'sergeev'), так как в первом сеансе транзакция еще не закомичена, а текущий уровень изоляции **read committed**.
Теперь завершим транзакцию в первом сеансе:
```
postgres=*# commit
postgres-*# ;
COMMIT
```
Делаем тот же select во втором сеансе, и видим нашу строку   **3 | sergey     | sergeev** , опять же, так как уровень изоляции **read committed**.
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Завершим транзакциб во втором сеансе
```
postgres=*# commit;
COMMIT
```
Начнем новую транзакцию во втором сеансе с уровнем изоляции **repeatable read**
```
postgres=# set transaction isolation level repeatable read;
SET
```
В первой сессии добавим новую запись **sveta      | svetova**
```
postgres=# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
Сделаем **select * from persons** во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Естественно записи мы еще не видим
Завершим транзакцию в первом сеансе
```
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
И посмотрим еще раз, что изменилось во втором сеансе.
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
И опять, мы не видим новой записи. Потому что мы установили уровень изоляции **repeatable read** и видим только те данные, которые были закоммичены на момент начала нашей транзакции.
Завершим ее, и в новой транзакции убедимся, что мы видим все данные.
```
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Done.

