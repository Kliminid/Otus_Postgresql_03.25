
1. Создал ВМ с помощью Virtual box

2. В настройках Virtual box в разделе "сеть" заменил тип подключения с NAT на Сетевой мост,чтобы можно было подключаться по ssh

3. Зашел от рута и установил утилиту openssh-server

4. Затем по ssh подключился к ВМ с помощью MobaXterm(Для удобства)

5. Установил docker
``` 
apt  update
apt install docker
apt install docker.io
```
6. Создал каталог /var/lib/postgres 
```
mkdir /var/lib/postgres
```
7. Развернул контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql и проверил,что контейнер запустился
```
docker run --name postgres-server -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=12345 -p 5432:5432 -d postgres:15
docker ps
```
8. Развернул контейнер с клиентом postgres,подключился к серверу после ввода пароля и сделал таблицу с парой строк
```
docker run --name postgres-client2 --network="host" -it postgres:15 psql -h localhost -U postgres -W
CREATE TABLE homework (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255)
);
INSERT INTO homework (name) VALUES ('Возраст');
INSERT INTO homework (name) VALUES ('Город');
```
9. Также подключился к контейнеру с другой вм. Для этого пришлось установить на новой вм клиент postgres + на первой вм отредактировать hba файл и сделать релоад,чтоб я смог подключиться.
10. Далее удалил контейнер с сервером и создал его заново
```
docker stop postgres-server
docker rm postgres-server
docker run --name postgres-server -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=12345 -p 5432:5432 -d postgres:15
```
11. Подключился снова из контейнера с клиентом к контейнеру с сервером
```
docker run --name postgres-client3 --network="host" -it postgres:15 psql -h localhost -U postgres -W
```
12. Проверил,что данные на месте
``` 
select * from homework;
 id |  name
----+---------
  1 | Возраст
  2 | Город
(2 rows)
```
