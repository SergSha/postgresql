<h3>### PostgreSQL ###</h3>

<p>репликация postgres</p>

<h4>Описание домашнего задания</h4>

<ul>
<li>настроить hot_standby репликацию с использованием слотов</li>
<li>настроить правильное резервное копирование</li>
</ul>
<ul>Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть
<li>Vagranfile (2 машины)</li>
<li>плейбук Ansible</li>
<li>конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,</li>
<li>конфиг barman, либо скрипт резервного копирования.</li>
</ul>
<p>Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием.</p>
<p>Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.</p>

<h4>Создание стенда PostgreSQL replication</h4>

<p>Содержимое Vagrantfile:</p>

<pre>[user@localhost postgresql]$ <b>vi ./Vagrantfile</b></pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :master => {
    :box_name => "centos/7",
    :vm_name => "master",
    :ip => '192.168.50.10',
    :mem => '1048'
  },
  :replica => {
    :box_name => "centos/7",
    :vm_name => "replica",
    :ip => '192.168.50.11',
    :mem => '1048'
  },
  :backup => {
    :box_name => "centos/7",
    :vm_name => "backup",
    :ip => '192.168.50.12',
    :mem => '1048'
  }
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", boxconfig[:mem]]
      end
#      if boxconfig[:vm_name] == "backup"
#        box.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/playbook.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.become = true
#          ansible.verbose = "vvv"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#        end
#      end
    end
  end
end</pre>

<p>Запустим виртуальные машины:</p>

<pre>[user@localhost postgresql]$ <b>vagrant up</b></pre>

<pre>[user@localhost postgresql]$ <b>vagrant status</b>
Current machine states:

master                    running (virtualbox)
replica                   running (virtualbox)
backup                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost postgresql]$</pre>

<h4>Физическая репликация</h4>

<h4>Сервер master</h4>

<p>Подключаемся по ssh к серверу master и зайдём с правами root:</p>

<pre>[user@localhost postgresql]$ <b>vagrant ssh master</b>
[vagrant@master ~]$ <b>sudo -i</b>
[root@master ~]#</pre>

<p>Подключаем репозиторий PostreSQL последней версии:</p>

<pre>[root@master ~]# <b>yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm</b>
...
Installed:
  pgdg-redhat-repo.noarch 0:42.0-28

Complete!
[root@master ~]#</pre>

<pre>[root@master ~]# <b>ls -l /etc/yum.repos.d/</b>
total 48
-rw-r--r--. 1 root root  1664 Apr  7  2020 CentOS-Base.repo
-rw-r--r--. 1 root root  1309 Apr  7  2020 CentOS-CR.repo
-rw-r--r--. 1 root root   649 Apr  7  2020 CentOS-Debuginfo.repo
-rw-r--r--. 1 root root   630 Apr  7  2020 CentOS-Media.repo
-rw-r--r--. 1 root root  1331 Apr  7  2020 CentOS-Sources.repo
-rw-r--r--. 1 root root  7577 Apr  7  2020 CentOS-Vault.repo
-rw-r--r--. 1 root root   314 Apr  7  2020 CentOS-fasttrack.repo
-rw-r--r--. 1 root root   616 Apr  7  2020 CentOS-x86_64-kernel.repo
<b>-rw-r--r--. 1 root root 11855 Oct 16 07:00 pgdg-redhat-all.repo</b>
[root@master ~]#</pre>

<p>Устанавливаем postgresql14-server:</p>

<pre>[root@master ~]# <b>yum install -y postgresql14-server</b>
...
Installed:
  postgresql14-server.x86_64 0:14.5-1PGDG.rhel7

Dependency Installed:
  postgresql14.x86_64 0:14.5-1PGDG.rhel7  postgresql14-libs.x86_64 0:14.5-1PGDG.rhel7 

Complete!
[root@master ~]#</pre>

<p>Инициализируем базы:</p>

<pre>[root@master ~]# <b>postgresql-14-setup initdb</b>
Initializing database ... OK

[root@master ~]#</pre>

<p>Запускаем сервис postresql:</p>

<pre>[root@master ~]# <b>systemctl enable postgresql-14 --now</b>
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql-14.service to /usr/lib/systemd/system/postgresql-14.service.
[root@master ~]#</pre>

<pre>[root@master ~]# <b>systemctl status postgresql-14</b>
● postgresql-14.service - PostgreSQL 14 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-14.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-11-06 10:49:43 UTC; 32s ago
     Docs: https://www.postgresql.org/docs/14/static/
  Process: 22378 ExecStartPre=/usr/pgsql-14/bin/postgresql-14-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 22383 (postmaster)
   CGroup: /system.slice/postgresql-14.service
           ├─22383 /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
           ├─22385 postgres: logger 
           ├─22387 postgres: checkpointer 
           ├─22388 postgres: background writer 
           ├─22389 postgres: walwriter 
           ├─22390 postgres: autovacuum launcher 
           ├─22391 postgres: stats collector 
           └─22392 postgres: logical replication launcher 

Nov 06 10:49:43 master systemd[1]: Starting PostgreSQL 14 database server...
Nov 06 10:49:43 master postmaster[22383]: 2022-11-06 10:49:43.431 UTC [22383] LOG:...ss
Nov 06 10:49:43 master postmaster[22383]: 2022-11-06 10:49:43.431 UTC [22383] HINT...".
Nov 06 10:49:43 master systemd[1]: Started PostgreSQL 14 database server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@master ~]#</pre>

<p>Задаем пароль для пользователя postgres:</p>

<pre>[root@master ~]# <b>passwd postgres</b>             # 'psql@Otus1234'
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@master ~]#</pre>

<p>Заходим в систему под данной учетной записью:</p>

<pre>[root@master ~]# <b>su - postgres</b>
-bash-4.2$</pre>

<p>Подключаемся к базе:</p>

<pre>-bash-4.2$ <b>psql</b>
psql (14.5)
Type "help" for help.

postgres=#</pre>

<p>Делаем тестовый запрос на получение списка таблиц:</p>

<pre>postgres=# <b>\dt *</b>
                    List of relations
   Schema   |          Name           | Type  |  Owner
------------+-------------------------+-------+----------
 pg_catalog | pg_aggregate            | table | postgres
 pg_catalog | pg_am                   | table | postgres
...
 pg_catalog | pg_user_mapping         | table | postgres
(62 rows)

postgres=#</pre>

<p>Чтобы выйти из оболочки psql:</p>

<pre>postgres=# <b>\q</b>
-bash-4.2$</pre>

<p>Отключиться от системы пользователем postgres:</p>

<pre>-bash-4.2$ <b>exit</b>
logout
[root@master ~]#</pre>

<p>Снова подключаемся к базе под пользователем postgres:</p>

<pre>[root@master ~]# <b>sudo -u postgres psql</b>
could not change directory to "/root": Permission denied
psql (14.5)
Type "help" for help.

postgres=#</pre>

<pre>postgres=# <b>select pg_is_in_recovery();</b>
 pg_is_in_recovery 
-------------------
 f
(1 row)

postgres=#</pre>

<p>Информация про слоты репликации:</p>

<pre>postgres=# <b>select * from pg_stat_replication;</b>
 pid | usesysid | usename | application_name | client_addr | client_hostname | client_p
ort | backend_start | backend_xmin | state | sent_lsn | write_lsn | flush_lsn | replay_
lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state | reply_time 
-----+----------+---------+------------------+-------------+-----------------+---------
----+---------------+--------------+-------+----------+-----------+-----------+--------
----+-----------+-----------+------------+---------------+------------+------------
(0 rows)

postgres=#</pre>

<p>Создадим базу replica:</p>

<pre>postgres=# <b>create database replica;</b>
CREATE DATABASE
postgres=#</pre>

<p>Выводим список баз:</p>

<pre>postgres=# <b>\l</b>
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 <b>replica   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | </b>
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=#</pre>

<p>Чтобы подключиться к базе replica:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=# \q
[root@master ~]#</pre>


<p>По умолчанию, сервер баз данных postresql разрешает подключение только с локального компьютера.<br />
Для начала посмотрим путь расположения конфигурационного файла postgresql.conf:</p>

<pre>[root@master ~]# <b>su - postgres -c "psql -c 'SHOW config_file;'"</b>
              config_file               
----------------------------------------
 /var/lib/pgsql/14/data/postgresql.conf
(1 row)

[root@master ~]#</pre>

<p>Открываем на редактирование основной файл конфигурации postgresql.conf:</p>

<pre>[root@master ~]# <b>vi /var/lib/pgsql/14/data/postgresql.conf</b></pre>

<p>Находим и редактируем следующие строки:</p>

<pre>#listen_addresses = 'localhost'</pre>

<p>на:</p>

<pre>listen_addresses = '192.168.50.10'</pre>

<p>Открываем на редактирование следующий конфигурационный файл pg_hba.conf:</p>

<pre>[root@master ~]# <b>vi /var/lib/pgsql/14/data/pg_hba.conf</b></pre>

<p>Находим и редактируем следующие строки:</p>

<pre># TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
<b>host    all             all             127.0.0.1/32            scram-sha-256</b>
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
<b>host    replication     all             127.0.0.1/32            scram-sha-256</b>
host    replication     all             ::1/128                 scram-sha-256</pre>

<p>на:</p>

<pre># TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
<b>host    all             all             192.168.50.11/32        scram-sha-256</b>
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
<b>host    replication     all             192.168.50.11/32        scram-sha-256</b>
host    replication     all             ::1/128                 scram-sha-256</pre>

<p>Перезапускаем сервис postgresql:</p>

<pre>[root@master ~]# <b>systemctl restart postgresql-14</b>
[root@master ~]#</pre>

<p>Задаем пароль для пользователя postgres:</p>

<pre>[root@replica ~]# <b>passwd postgres</b>             # 'psql@Otus1234'
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@replica ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql -c "ALTER ROLE postgres PASSWORD 'psql@Otus1234'"</b>
could not change directory to "/root": Permission denied
ALTER ROLE
[root@master ~]#</pre>

<p>Снова заходим в postgres:</p>

<pre>[root@master ~]# <b>sudo -u postgres psql</b>
could not change directory to "/root": Permission denied
psql (14.5)
Type "help" for help.

postgres=#</pre>

<p>Смотрим слот репликации:</p>

<pre>postgres=# <b>select * from pg_stat_replication;</b>
 pid | usesysid | usename | application_name | client_addr | client_hostname | client_p
ort | backend_start | backend_xmin | state | sent_lsn | write_lsn | flush_lsn | replay_
lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state | reply_time 
-----+----------+---------+------------------+-------------+-----------------+---------
----+---------------+--------------+-------+----------+-----------+-----------+--------
----+-----------+-----------+------------+---------------+------------+------------
(0 rows)

postgres=#</pre>

<p>Как мы видим, таблица пустая.</p>

<p>Выводим список имеющихся баз данных:</p>

<pre>postgres=# <b>\l</b>
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileg
es
-----------+----------+----------+-------------+-------------+------------------
-----
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
(3 rows)

postgres=#</pre>

<p>Создадим базу данных replica:</p>

<pre>postgres=# <b>create database replica;</b>
CREATE DATABASE
postgres=#</pre>

<p>Выводим ещё раз список имеющихся баз данных:</p>

<pre>postgres=# <b>\l</b>
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileg
es
-----------+----------+----------+-------------+-------------+------------------
-----
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 <b>replica</b>   | <b>postgres</b> | <b>UTF8</b>     | <b>en_US.UTF-8</b> | <b>en_US.UTF-8</b> | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
(4 rows)

postgres=#</pre>

<p>Как мы видим, что добавилась новая база данных replica.<br />
Подключимся к созданной базе replica:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Создадим таблицу t с полем t в формате int:</p>

<pre>replica=# <b>replica=# create table cars (id int,name varchar);</b>
CREATE TABLE
replica=#</pre>

<p>В эту таблицу добавим запись:</p>

<pre>replica=# <b>insert into cars(id,name) values(1,'Niva');</b>
INSERT 0 1
replica=#</pre>

<p>Убедимся, что в таблице cars появилась новая запись:</p>

<pre>replica=# <b>select * from cars;</b>
 id | name 
----+------
  1 | Niva
(1 row)

replica=#</pre>



<h4>Сервер replica</h4>

<p>В отдельном окне терминала подключимся к серверу replica и зайдём под пользователем root:</p>

<pre>[user@localhost postgresql]$ <b>vagrant ssh replica</b>
[vagrant@replica ~]$ <b>sudo -i</b>
[root@replica ~]#</pre>

<p>Также, как и на сервере master, подключим репозиторий PostreSQL последней версии и установим пакет postgreSQL:</p>

<pre>[root@replica ~]# <b>yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm</b></pre>

<pre>[root@replica ~]# <b>yum install -y postgresql14-server</b></pre>

<p>Задаем пароль для пользователя postgres:</p>

<pre>[root@replica ~]# <b>passwd postgres</b>             # 'psql@Otus1234'
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@replica ~]#</pre>

<p>Удалим директорий postgresql:</p>

<pre>[root@replica ~]# <b>rm -rf /var/lib/pgsql/14/data/</b></pre>

<p>Подключаемся к базе данных на сервере master:</p>

<pre>[root@replica ~]# <b>sudo -u postgres pg_basebackup -h 192.168.50.10 -R -D /var/lib/pgsql/14/data -U postgres -W</b>
could not change directory to "/root": Permission denied
Password:
[root@replica ~]#</pre>

<p>Запускаем сервис postresql:</p>

<pre>[root@replica ~]# <b>systemctl enable postgresql-14 --now</b>
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql-14.service to /usr/lib/systemd/system/postgresql-14.service.
[root@replica ~]#</pre>

<pre>[root@replica ~]# <b>systemctl status postgresql-14</b>
● postgresql-14.service - PostgreSQL 14 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-14.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-11-07 12:22:52 UTC; 32s ago
     Docs: https://www.postgresql.org/docs/14/static/
  Process: 3757 ExecStartPre=/usr/pgsql-14/bin/postgresql-14-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 3762 (postmaster)
   CGroup: /system.slice/postgresql-14.service
           ├─3762 /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
           ├─3764 postgres: logger
           ├─3765 postgres: startup recovering 000000010000000000000003
           ├─3766 postgres: checkpointer
           ├─3767 postgres: background writer
           ├─3768 postgres: stats collector
           └─3769 postgres: walreceiver streaming 0/3000148

Nov 07 12:22:48 replica systemd[1]: Starting PostgreSQL 14 database server...
Nov 07 12:22:48 replica postmaster[3762]: 2022-11-07 12:22:48.632 UTC [3762]...s
Nov 07 12:22:48 replica postmaster[3762]: 2022-11-07 12:22:48.632 UTC [3762]....
Nov 07 12:22:52 replica systemd[1]: Started PostgreSQL 14 database server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@replica ~]#</pre>



<h4>Проверка работы репликации</h4>

<p>Заходим в систему под данной учетной записью и подключаемся к базе:</p>

<pre>[root@replica ~]# <b>sudo -u postgres psql</b>
could not change directory to "/root": Permission denied
psql (14.5)
Type "help" for help.

postgres=#</pre>

<p>Выводим список баз данных:</p>

<pre>postgres=# <b>\l</b>
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileg
es
-----------+----------+----------+-------------+-------------+------------------
-----
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 <b>replica</b>   | <b>postgres</b> | <b>UTF8</b>     | <b>en_US.UTF-8</b> | <b>en_US.UTF-8</b> |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
(4 rows)

postgres=#</pre>

<p>Подключаемся к базе данных replica:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Выводим данные из таблицы cars:</p>

<pre>replica=# <b>select * from cars;</b>
 id | name 
----+------
  1 | Niva
(1 row)

replica=#</pre>

<p>На сервере master в таблице cars добавим ещё одну запись:</p>

<pre>replica=# <b>insert into cars(id,name) values(2,'Lada');</b>
INSERT 0 1
replica=# <b>select * from cars;</b>
 id | name 
----+------
  1 | Niva
  2 | Lada
(2 rows)

replica=#</pre>

<p>Убедимся, что на сервере replica внеслись изменения:</p>

<pre>replica=# <b>select * from cars;</b>
 id | name 
----+------
  1 | Niva
  2 | Lada
(2 rows)

replica=#</pre>

<p>Как мы видим, что на сервере replica в таблице cars также появилась вторая запись.</p>

<p>Теперь попробуем добавить запись в таблицу cars на самом сервере replica:</p>

<pre>replica=# <b>INSERT INTO cars(id,name) VALUES(3,'Volga');</b>
ERROR:  cannot execute INSERT in a read-only transaction
replica=#</pre>

<p>Как видим, на сервере replica не разрешает запись.</p>

<h4>Логическая репликация</h4>

<p>Перенастроим наш сервер master на логическую репликацию. Для этого в конфиг файле postgesql.conf:</p>

<pre>[root@master ~]# vi /var/lib/pgsql/14/data/postgresql.conf</pre>

<p>внесём изменение в строке:</p>

<pre>#wal_level = replica                    # minimal, replica, or logical</pre>

<p>на:</p>

<pre>wal_level = logical                    # minimal, replica, or logical</pre>

<p>Перезапускаем postgresql сервис:</p>

<pre>[root@master ~]# systemctl restart postgresql-14
[root@master ~]#</pre>

<p>Снова подключимся к базе данных replica:</p>

<pre>[root@master ~]# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.5)
Type "help" for help.

postgres=# \c replica
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Добавим новую таблицу cities:</p>

<pre>replica=# CREATE TABLE cities(id INT,name VARCHAR);
CREATE TABLE
replica=#</pre>

<p>Создаём публикацию для таблицы cities:</p>

<pre>replica=# CREATE PUBLICATION cities_pub FOR TABLE cities;
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to logical before creating subscriptions.
CREATE PUBLICATION
replica=#</pre>

<p>Смотрим, что получилось:</p>

<pre>replica=# select * from pg_publication;
  oid  |  pubname   | pubowner | puballtables | pubinsert | pubupdate | pubdelet
e | pubtruncate | pubviaroot
-------+------------+----------+--------------+-----------+-----------+---------
--+-------------+------------
 16395 | cities_pub |       10 | f            | t         | t         | t
  | t           | f
(1 row)

replica=#</pre>

<pre>replica=# select * from pg_publication_tables;
  pubname   | schemaname | tablename
------------+------------+-----------
 cities_pub | public     | cities
(1 row)

replica=#</pre>

<p>Добавим запись в таблицу cities:</p>

<pre>replica=# INSERT INTO cities(id,name) VALUES(1,'Moscow');
INSERT 0 1
replica=# SELECT * FROM cities;
 id |  name
----+--------
  1 | Moscow
(1 row)

replica=#</pre>

<p>На сервере replica перезапустим postgresql сервис:</p>

<pre>[root@replica ~]# sudo -u postgres /usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data promote
could not change directory to "/root": Permission denied
waiting for server to promote..... done
server promoted
[root@replica ~]#</pre>

<pre>[root@replica ~]# systemctl restart postgresql-14
[root@replica ~]#</pre>

<p>Создадим подписку к базе данных по порту, пользователю и паролю:</p>

<pre>replica=# CREATE SUBSCRIPTION cities_sub
replica-# CONNECTION 'host=192.168.50.10 port=5432 user=postgres password=psql@Otus1234 dbname=replica'
replica-# PUBLICATION cities_pub;
NOTICE:  created replication slot "cities_sub" on publisher
CREATE SUBSCRIPTION
replica=#</pre>

<pre>replica=# SELECT * FROM PG_REPLICATION_SLOTS;
 slot_name | plugin | slot_type | datoid | database | temporary | active | activ
e_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | s
afe_wal_size | two_phase
-----------+--------+-----------+--------+----------+-----------+--------+------
------+------+--------------+-------------+---------------------+------------+--
-------------+-----------
(0 rows)

replica=#</pre>

<pre>replica=# SELECT * FROM PG_SUBSCRIPTION;
  oid  | subdbid |  subname   | subowner | subenabled | subbinary | substream |
                                  subconninfo
 | subslotname | subsynccommit | subpublications
-------+---------+------------+----------+------------+-----------+-----------+-
--------------------------------------------------------------------------------
-+-------------+---------------+-----------------
 24584 |   16384 | cities_sub |       10 | t          | f         | f         |
host=192.168.50.10 port=5432 user=postgres password=psql@Otus1234 dbname=replica
 | cities_sub  | off           | {cities_pub}
(1 row)

replica=#</pre>

<p>Смотрим записи в таблице cities:</p>

<pre>replica=# SELECT * FROM cities;
 id |  name
----+--------
  1 | Moscow
(1 row)

replica=#</pre>

<pre>replica=# SELECT * FROM PG_STAT_SUBSCRIPTION;
 subid |  subname   |  pid  | relid | received_lsn |      last_msg_send_time
   |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time

-------+------------+-------+-------+--------------+----------------------------
---+-------------------------------+----------------+---------------------------
----
 24584 | cities_sub | 22962 |       | 0/5020750    | 2022-11-08 09:09:30.787324+
00 | 2022-11-08 09:09:30.786033+00 | 0/5020750      | 2022-11-08 09:09:30.787324
+00
(1 row)

replica=#</pre>

<p>Попробуем на сервере master добавить ещё одну запись в таблицу cities:</p>

<pre>replica=# INSERT INTO cities(id,name) Values(2,'Madrid');
INSERT 0 1
replica=#</pre>

<p>На сервере replica выводим данные таблицы cities:</p>

<pre>replica=# SELECT * FROM cities;
 id |  name
----+--------
  1 | Moscow
  2 | Madrid
(2 rows)

replica=#</pre>

<p>Как мы видим, при логической репликации данные на сервере replica также изменяются в соответствии с изменениями на сервере master.</p>

<p>Чтобы отменить логическую репликацию, на сервере master нужно выполнить следующие команды:</p>

<pre>DROP PUBLICATION cities_pub;</pre>

<pre>vi /var/lib/pgsql/14/data/postgresql.conf
#wal_level = replica                     # minimal, replica, or logical</pre>





