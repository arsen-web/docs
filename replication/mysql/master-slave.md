### **Как настроить MySQL Master-Slave репликацию?**
##### Шаг 1. Настройка Master
Сервер (IP - 10.10.0.1) который будет выступать как Master, необходимо внести правки в my.cnf:

Путь до файла my.cnf
* Для ubuntu /etc/mysql/mysql.conf.d/mysqld.cnf
* Для MacOS /usr/local/etc/my.cnf

Выбираем ID сервера: (произвольное число, лучше начинать с 1) <br /> 
`server-id = 1`

Путь к бинарному логу: <br />
`log_bin = /var/log/mysql/mysql-bin.log`

Название Вашей базы данных, которая будет реплицироваться: <br />
`binlog_do_db = <имя базы данных>`

Настройка формата бинарного файла:
* STATEMENT - пишет в бинлог по сути sql запросы.
* ROW - пишет в бинлог измененные строки.
* MIXED - пишет промежуточный формат, который старается использовать STATEMENT, когда возможно, а когда нет — ROW. 

`binlog_format = ROW`

Определяем размер файла бинлога, который используется при репликации: <br />
`max_binlog_size = 500M`

После какого кол-во дней автоматически удалять файлы бинлогов: <br />
`expire_logs_days = 7` 

Определяем логику синхронизации данных из бинлога с диском:
Если значение равно 1, запись на диск будет происходить после каждой транзакции.
Это делает хранилище очень надежным, но крайне сильно нагружает дисковую подсистему на мастере.
Значение 0 отключит синхронизацию из Mysql, 
и база данных будет полагаться на ОС в вопросе записи лога на диск.
Такое значение может увеличить производительность мастера в несколько раз:  <br />
`sync_binlog = 1`

Хранение временный файлов: <br />
`tmpdir = /tmp`

Сохраняем файл и перезапускаем MySql: <br />
`sudo service mysql restart`
<br /><br />
##### Шаг 2. Права на репликацию
Для этого запускаем MySql в консоле Master: <br />
`mysql -u root -p`

Необходимо создать профиль пользователя, из под которого будет происходить репликация: <br />
* Для MySql 8: <br /> 
`CREATE USER '<ИМЯ_ПОЛЬЗОВАТЕЛЯ>'@'<ИМЯ_ХОСТА>' IDENTIFIED WITH mysql_native_password BY '<ПАРОЛЬ>';`
* Для MySql 5.7: <br /> 
`CREATE USER '<ИМЯ_ПОЛЬЗОВАТЕЛЯ>'@'<ИМЯ_ХОСТА>' IDENTIFIED BY '<ПАРОЛЬ>';`

Назначаем права пользователю для репликации: <br />
`GRANT REPLICATION SLAVE ON *.* TO '<ИМЯ_ПОЛЬЗОВАТЕЛЯ>'@'<ИМЯ_ХОСТА>';`

Обновляем права доступа: <br />
`FLUSH PRIVILEGES;`

Блокируем все таблицы в нашей базе данных: <br />
`USE <ИМЯ_БАЗЫ>` <br />
`FLUSH TABLES WITH READ LOCK;`

Проверяем статус Master сервера: <br />
`SHOW MASTER STATUS;`

Мы увидим что-то похожее на: <br />
![mountains](./img/replication3.png)
<br /><br />
##### Шаг 3. Дамп базы
Необходимо сделать дамп базы данных: <br />
`mysqldump -u <ИМЯ_ПОЛЬЗОВАТЕЛЯ> -p <ИМЯ_БАЗЫ> > data-dump.sql`

Разблокируем таблицы в консоли mysql: <br />
`UNLOCK TABLES;`
<br /><br />
##### Шаг 4. Создание базы на Slave
Для этого запускаем MySql в консоле Slave: <br />
`mysql -u root -p`

В консоли mysql на Slave создаем базу с таким же именем, как и на Master: <br />
`CREATE DATABASE <ИМЯ_БАЗЫ> CHARACTER SET utf8 COLLATE utf8_general_ci;`

После этого загружаем дамп: <br />
`mysql -u <ИМЯ_ПОЛЬЗОВАТЕЛЯ> -p <ИМЯ_БАЗЫ> < data-dump.sql`
<br /><br />
##### Шаг 5. Настройка Slave
Сервер (IP - 10.10.0.2) который будет выступать как Slave, необходимо внести правки в my.cnf:

Путь до файла my.cnf
* Для ubuntu /etc/mysql/mysql.conf.d/mysqld.cnf
* Для MacOS /usr/local/etc/my.cnf

Выбираем ID сервера: (удобно выбирать следующим числом после Master) <br /> 
`server-id = 2`

Путь к relay логу: <br />
`relay-log = /var/log/mysql/mysql-relay-bin.log`

Путь к bin логу на Master: <br />
`log_bin = /var/log/mysql/mysql-bin.log`

База данных для репликации: <br />
`binlog_do_db = <имя базы данных>`

Сохраняем файл и перезапускаем MySql: <br />
`sudo service mysql restart`
<br /><br />
##### Шаг 6. Запуск Slave
В консоли mysql на Slave необходимо выполнить запрос: <br />
`CHANGE MASTER TO 
        MASTER_HOST='<ХОСТ>', 
        MASTER_USER='<ИМЯ_ПОЛЬЗОВАТЕЛЯ>', 
        MASTER_PASSWORD='<ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ>',
        MASTER_LOG_FILE = '<>', 
        MASTER_LOG_POS = <>;` 
<br />
**MASTER_LOG_FILE** и **MASTER_LOG_POS** берем из Master командой `SHOW MASTER STATUS;` <br />
![mountains](./img/replication3.png)

После этого запускаем репликацию на Slave: <br />
`START SLAVE;`

Проверить работу репликации на Slave можно запросом: <br />
`SHOW SLAVE STATUS\G`

![mountains](./img/replication4.png)
***
