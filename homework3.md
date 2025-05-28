
1. Создал ВМ с помощью Virtual box

2. В настройках Virtual box в разделе "сеть" заменил тип подключения с NAT на Сетевой мост,чтобы можно было подключаться по ssh

3. Зашел от рута и установил утилиту openssh-server

4. Затем по ssh подключился к ВМ с помощью MobaXterm(Для удобства)

5. Установил PostgreSQL 


6. Проверил запущен ли кластер (Запущен)
```
 sudo -u postgres pg_lsclusters
 ```

 7. Зашел в psql и сделал произвольную таблицу с произвольным содержимым
 ```
create table homework(c1 text);
insert into homework values('1');
```
8. Остановил postgres 
``` 
sudo -u postgres pg_ctlcluster 15 main stop
systemctl stop postgresql@15-main
```
9. Cоздал новый виртуальный диск в Virtual box и добавил его к существующей вм, перед этим отключил вм

10. Проинициализировал диск согласно инструкции и подмонтировал файловую систему
```
fdisk /dev/sdb
потом создал раздел /dev/sdb1  и сохранил нажав на w
mkfs.ext4 /dev/sdb1
mkdir /mnt/data
mount /dev/sdb1 /mnt/data
и проверил примонтировался ли командой df -h
```
11.После ребут диск не остался примонтированным,поэтому пришлось работать с fstab
```
blkid /dev/sdb1
Копируем uuid и заходим в /etc/fstab и вставляем строчку:UUID=55184734-f94b-4f19-935f-367fc-8815330 /mnt/data ext4 defaults 0 2
сохраняем
 ```
 12. Ребутаем вм и проверяем примонтировался ли диск (примонтировался)

 13. Сделал пользователя postgres владельцем /mnt/data
 ```
 chown -R postgres:postgres /mnt/data
```
14. Перенес содержимое /var/lib/postgres/15 в /mnt/data
```
mv /var/lib/postgresql/15 /mnt/data
```
15.Кластер запустить не получится так как надо изменить конфигурационный файл,а именно строчку data_directory, потому что мы перенесли все в /mnt/data

16.После этого делаем рестарт postgresql и проверяем статус (все работает)
```
systemctl restart postgresql 
systemctl status postgresql
```
17.Заходим в psql и проверяем содержимое таблицы командой select * from homework; (цифра 1,которую я вставлял в таблицу сохранилась)