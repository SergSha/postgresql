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

<p>Подключаемся по ssh к серверу <i>master</i> и зайдём с правами <i>root</i>:</p>

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

<p>Задаем пароль для пользователя <i>postgres</i>:</p>

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

<p>Снова подключаемся к базе под пользователем <i>postgres</i>:</p>

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

<p>Создадим базу <i>replica</i>:</p>

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

<p>Чтобы подключиться к базе <i>replica</i>:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=# \q
[root@master ~]#</pre>


<p>По умолчанию, сервер баз данных postresql разрешает подключение только с локального компьютера.<br />
Для начала посмотрим путь расположения конфигурационного файла <i>postgresql.conf</i>:</p>

<pre>[root@master ~]# <b>su - postgres -c "psql -c 'SHOW config_file;'"</b>
              config_file               
----------------------------------------
 /var/lib/pgsql/14/data/postgresql.conf
(1 row)

[root@master ~]#</pre>

<p>Открываем на редактирование основной файл конфигурации <i>postgresql.conf</i>:</p>

<pre>[root@master ~]# <b>vi /var/lib/pgsql/14/data/postgresql.conf</b></pre>

<p>Находим и редактируем следующие строки:</p>

<pre>#listen_addresses = 'localhost'</pre>

<p>на:</p>

<pre>listen_addresses = '*'</pre>

<p>Открываем на редактирование следующий конфигурационный файл <i>pg_hba.conf</i>:</p>

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
<b>host    all             all             192.168.50.0/24        scram-sha-256</b>
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
<b>host    replication     all             192.168.50.0/24        scram-sha-256</b>
host    replication     all             ::1/128                 scram-sha-256</pre>

<p>Перезапускаем сервис postgresql:</p>

<pre>[root@master ~]# <b>systemctl restart postgresql-14</b>
[root@master ~]#</pre>

<p>Задаем пароль для пользователя <i>postgres</i>:</p>

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

<pre>postgres=# <b>SELECT * FROM pg_stat_replication;</b>
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

<p>Создадим базу данных <i>replica</i>:</p>

<pre>postgres=# <b>CREATE DATABASE replica;</b>
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

<p>Как мы видим, что добавилась новая база данных <i>replica</i>.<br />
Подключимся к созданной базе <i>replica</i>:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Создадим таблицу <i>cars</i> с полями <i>id</i> и <i>name</i>:</p>

<pre>replica=# replica=# <b>CREATE TABLE cars (id INT,name VARCHAR);</b>
CREATE TABLE
replica=#</pre>

<p>В эту таблицу добавим запись:</p>

<pre>replica=# <b>INSERT INTO cars(id,name) VALUES(1,'Niva');</b>
INSERT 0 1
replica=#</pre>

<p>Убедимся, что в таблице <i>cars</i> появилась новая запись:</p>

<pre>replica=# <b>SELECT * FROM cars;</b>
 id | name 
----+------
  1 | Niva
(1 row)

replica=#</pre>



<h4>Сервер replica</h4>

<p>В отдельном окне терминала подключимся к серверу <i>replica</i> и зайдём под пользователем <i>root</i>:</p>

<pre>[user@localhost postgresql]$ <b>vagrant ssh replica</b>
[vagrant@replica ~]$ <b>sudo -i</b>
[root@replica ~]#</pre>

<p>Также, как и на сервере <i>master</i>, подключим репозиторий PostreSQL последней версии и установим пакет postgreSQL:</p>

<pre>[root@replica ~]# <b>yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm</b></pre>

<pre>[root@replica ~]# <b>yum install -y postgresql14-server</b></pre>

<p>Задаем пароль для пользователя <i>postgres</i>:</p>

<pre>[root@replica ~]# <b>passwd postgres</b>             # 'psql@Otus1234'
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@replica ~]#</pre>

<p>Удалим директорий postgresql:</p>

<pre>[root@replica ~]# <b>rm -rf /var/lib/pgsql/14/data/</b></pre>

<p>Подключаемся к базе данных на сервере <i>master</i>:</p>

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

<p>Подключаемся к базе данных <i>replica</i>:</p>

<pre>postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Выводим данные из таблицы <i>cars</i>:</p>

<pre>replica=# <b>SELECT * FROM cars;</b>
 id | name 
----+------
  1 | Niva
(1 row)

replica=#</pre>

<p>На сервере <i>master</i> в таблице <i>cars</i> добавим ещё одну запись:</p>

<pre>replica=# <b>INSERT INTO cars(id,name) VALUES(2,'Lada');</b>
INSERT 0 1
replica=# <b>select * from cars;</b>
 id | name 
----+------
  1 | Niva
  2 | Lada
(2 rows)

replica=#</pre>

<p>Убедимся, что на сервере <i>replica</i> внеслись изменения:</p>

<pre>replica=# <b>SELECT * FROM cars;</b>
 id | name 
----+------
  1 | Niva
  2 | Lada
(2 rows)

replica=#</pre>

<p>Как мы видим, что на сервере <i>replica</i> в таблице <i>cars</i> также появилась вторая запись.</p>

<p>Теперь попробуем добавить запись в таблицу cars на самом сервере <i>replica</i>:</p>

<pre>replica=# <b>INSERT INTO cars(id,name) VALUES(3,'Volga');</b>
ERROR:  cannot execute INSERT in a read-only transaction
replica=#</pre>

<p>Как видим, на сервере <i>replica</i> не разрешает запись.</p>

<h4>Логическая репликация</h4>

<p>Перенастроим наш сервер <i>master</i> на логическую репликацию. Для этого в конфиг файле postgesql.conf:</p>

<pre>[root@master ~]# <b>vi /var/lib/pgsql/14/data/postgresql.conf</b></pre>

<p>внесём изменение в строке:</p>

<pre>#wal_level = replica                    # minimal, replica, or logical</pre>

<p>на:</p>

<pre>wal_level = logical                    # minimal, replica, or logical</pre>

<p>Перезапускаем postgresql сервис:</p>

<pre>[root@master ~]# <b>systemctl restart postgresql-14</b>
[root@master ~]#</pre>

<p>Снова подключимся к базе данных <i>replica</i>:</p>

<pre>[root@master ~]# <b>sudo -u postgres psql</b>
could not change directory to "/root": Permission denied
psql (14.5)
Type "help" for help.

postgres=# <b>\c replica</b>
You are now connected to database "replica" as user "postgres".
replica=#</pre>

<p>Добавим новую таблицу <i>cities</i>:</p>

<pre>replica=# <b>CREATE TABLE cities(id INT,name VARCHAR);</b>
CREATE TABLE
replica=#</pre>

<p>Создаём публикацию для таблицы <i>cities</i>:</p>

<pre>replica=# <b>CREATE PUBLICATION cities_pub FOR TABLE cities;</b>
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to logical before creating subscriptions.
CREATE PUBLICATION
replica=#</pre>

<p>Смотрим, что получилось:</p>

<pre>replica=# <b>SELECT * FROM pg_publication;</b>
  oid  |  pubname   | pubowner | puballtables | pubinsert | pubupdate | pubdelet
e | pubtruncate | pubviaroot
-------+------------+----------+--------------+-----------+-----------+---------
--+-------------+------------
 16395 | cities_pub |       10 | f            | t         | t         | t
  | t           | f
(1 row)

replica=#</pre>

<pre>replica=# <b>SELECT * FROM pg_publication_tables;</b>
  pubname   | schemaname | tablename
------------+------------+-----------
 cities_pub | public     | cities
(1 row)

replica=#</pre>

<p>Добавим запись в таблицу <i>cities</i>:</p>

<pre>replica=# <b>INSERT INTO cities(id,name) VALUES(1,'Moscow');</b>
INSERT 0 1
replica=# <b>SELECT * FROM cities;</b>
 id |  name
----+--------
  1 | Moscow
(1 row)

replica=#</pre>

<p>На сервере <i>replica</i> перезапустим postgresql сервис:</p>

<pre>[root@replica ~]# <b>sudo -u postgres /usr/pgsql-14/bin/pg_ctl -D /var/lib/pgsql/14/data promote</b>
could not change directory to "/root": Permission denied
waiting for server to promote..... done
server promoted
[root@replica ~]#</pre>

<pre>[root@replica ~]# <b>systemctl restart postgresql-14</b>
[root@replica ~]#</pre>

<p>Создадим подписку к базе данных по порту, пользователю и паролю:</p>

<pre>replica=# <b>CREATE SUBSCRIPTION cities_sub</b>
replica-# CONNECTION 'host=192.168.50.10 port=5432 user=postgres password=psql@Otus1234 dbname=replica'
replica-# PUBLICATION cities_pub;
NOTICE:  created replication slot "cities_sub" on publisher
CREATE SUBSCRIPTION
replica=#</pre>

<pre>replica=# <b>SELECT * FROM PG_REPLICATION_SLOTS;</b>
 slot_name | plugin | slot_type | datoid | database | temporary | active | activ
e_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | s
afe_wal_size | two_phase
-----------+--------+-----------+--------+----------+-----------+--------+------
------+------+--------------+-------------+---------------------+------------+--
-------------+-----------
(0 rows)

replica=#</pre>

<pre>replica=# <b>SELECT * FROM PG_SUBSCRIPTION;</b>
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

<p>Смотрим записи в таблице <i>cities</i>:</p>

<pre>replica=# <b>SELECT * FROM cities;</b>
 id |  name
----+--------
  1 | Moscow
(1 row)

replica=#</pre>

<pre>replica=# <b>SELECT * FROM PG_STAT_SUBSCRIPTION;</b>
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

<p>Попробуем на сервере <i>master</i> добавить ещё одну запись в таблицу <i>cities</i>:</p>

<pre>replica=# <b>INSERT INTO cities(id,name) Values(2,'Madrid');</b>
INSERT 0 1
replica=#</pre>

<p>На сервере <i>replica</i> выводим данные таблицы <i>cities</i>:</p>

<pre>replica=# <b>SELECT * FROM cities;</b>
 id |  name
----+--------
  1 | Moscow
  2 | Madrid
(2 rows)

replica=#</pre>

<p>Как мы видим, при логической репликации данные на сервере <i>replica</i> также изменяются в соответствии с изменениями на сервере <i>master</i>.</p>

<p>Чтобы отменить логическую репликацию, на сервере <i>master</i> нужно выполнить следующие команды:</p>

<pre><b>DROP PUBLICATION cities_pub;<b></pre>

<pre><b>vi /var/lib/pgsql/14/data/postgresql.conf<b>
#wal_level = replica                     # minimal, replica, or logical</pre>



<h4>Резервное копирование</h4>

<p>Для начала на сервере <i>master</i> создадим директорий /backup для резервного копирования БД postgresql:</p>

<pre>[root@master ~]# <b>mkdir /backup</b>
[root@master ~]# <b>chown -R postgres: /backup/</b>
[root@master ~]# <b>ls -ld /backup/</b>
drwxr-xr-x. 2 postgres postgres 6 Nov  9 10:09 /backup/
[root@master ~]#</pre>

<p>Затем на сервере же <i>master</i> для теста создадим базу данных <i>backup</i>:</p>

<pre>postgres=# <b>CREATE DATABASE backup;</b>
CREATE DATABASE
postgres=#</pre>

<p>Подключаемся к базе данных <i>backup</i>:</p>

<pre>postgres=# <b>\c backup</b>
You are now connected to database "backup" as user "postgres".
backup=#</pre>

<pre>backup=# <b>SELECT current_database();</b>
 current_database
------------------
 backup
(1 row)

backup=#</pre>

<p>Создадим таблицу <i>fruits</i> с полями <i>id</i>, <i>name</i> и <i>count</i>:</p>

<pre>backup=# <b>CREATE TABLE fruits(id INT,name TEXT,count INT);</b>
CREATE TABLE
backup=#</pre>

<p>Вставим несколько записей:</p>

<pre>backup=# <b>INSERT INTO fruits(id,name,count) VALUES(1,'apple',7);</b>
INSERT 0 1
backup=# <b>INSERT INTO fruits(id,name,count) VALUES(2,'pear',3);</b>
INSERT 0 1
backup=# <b>INSERT INTO fruits(id,name,count) VALUES(3,'banana',2);</b>
INSERT 0 1
backup=#</pre>

<p>Выводим список баз:</p>

<pre>postgres=# <b>\l</b>
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileg
es
-----------+----------+----------+-------------+-------------+------------------
-----
 <b>backup</b>    | <b>postgres</b> | <b>UTF8</b>     | <b>en_US.UTF-8</b> | <b>en_US.UTF-8</b> |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 replica   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/post
gres
(5 rows)

postgres=#</pre>

<p>Подключимся к базе данных <i>backup</i>:</p>

<pre>postgres=# <b>\c backup</b>
You are now connected to database "backup" as user "postgres".
backup=#</pre>

<p>Выводим таблицу backup:</p>

<pre>backup=# <b>SELECT * FROM fruits;</b>
 id |  name  | count
----+--------+-------
  1 | apple  |     7
  2 | pear   |     3
  3 | banana |     2
(3 rows)

backup=#</pre>

<p>Скопируем данные таблицы <i>fruits</i> в csv файл:</p>

<pre>backup=# <b>COPY fruits TO '/backup/fruits.csv' CSV HEADER;</b>
COPY 3
backup=#</pre>

<p>Выгруженный файл <i>fruits.csv</i>:</p>

<pre>[root@master ~]# <b>ls -l /backup/fruits.csv</b>
-rw-r--r--. 1 postgres postgres 44 Nov  9 12:49 /backup/fruits.csv
[root@master ~]#</pre>

<p>Содержимое файла <i>fruits.csv</i>:</p>

<pre>[root@master ~]# <b>cat /backup/fruits.csv</b>
id,name,count
1,apple,7
2,pear,3
3,banana,2
[root@master ~]#</pre>

<p>Для восстановления таблицы из csv файла нужно ОБЯЗАТЕЛЬНО создать новую таблицу. <br />
В нашем случае создадим таблицу <i>fruits2</i>. Создавать будем на сервере <i>master</i>, предварительно переместив csv файл:</p>

<pre>backup=# <b>CREATE TABLE fruits2(id INT,name TEXT,count INT);</b>
CREATE TABLE
backup=#</pre>

<p>Проверим, что создана пустая таблица <i>fruits2</i>:</p>

<pre>backup=# <b>SELECT * FROM fruits2;</b>
 id | name | count 
----+------+-------
(0 rows)

backup=#</pre>

<p>Загрузим в эту таблицу данные из csv файла <i>fruits.csv</i>:</p>

<pre>backup=# <b>COPY fruits2 FROM '/backup/fruits.csv' CSV HEADER;</b>
COPY 3
backup=#</pre>

<p>Проверим, что в таблицу <i>fruits2</i> загрузились данные с файла <i>fruits.csv</i>:</p>

<pre>backup=# <b>SELECT * FROM fruits2;</b>
 id |  name  | count 
----+--------+-------
  1 | apple  |     7
  2 | pear   |     3
  3 | banana |     2
(3 rows)

backup=#</pre>

<p>Как видим, нам удалось восстановить таблицу.</p>



<p>В отдельном окне терминала подключимся к серверу <i>backup</i> и зайдём под пользователем <i>root</i>:</p>

<pre>[user@localhost postgresql]$ <b>vagrant ssh backup</b>
[vagrant@backup ~]$ <b>sudo -i</b>
[root@backup ~]#</pre>

<p>Также, как и на серверах master и replica, подключим репозиторий PostreSQL последней версии и установим пакет postgreSQL:</p>

<pre>[root@backup ~]# <b>yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm</b></pre>

<pre>[root@backup ~]# <b>yum install -y postgresql14-server</b></pre>

<p>Задаем пароль для пользователя <i>postgres</i>:</p>

<pre>[root@backup ~]# <b>passwd postgres</b>             # 'psql@Otus1234'
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@backup ~]#</pre>

<p>Сначала также как и на сервере <i>replica</i> настроим репликацию. На практике для резервного копирования БД обычно подключаются к серверу репликации, чтобы не нагружать главный сервер. Мы для упрощения будем подключаться к серверу master.</p>

<p>Удалим директорий <i>postgresql</i>:</p>

<pre>[root@backup ~]# <b>rm -rf /var/lib/pgsql/14/data/</b>
[root@backup ~]#</pre>

<p>Подключаемся к базе данных на сервере <i>master</i>:</p>

<pre>[root@backup ~]# <b>sudo -u postgres pg_basebackup -h 192.168.50.10 -R -D /var/lib/pgsql/14/data -U postgres -W</b>
could not change directory to "/root": Permission denied
Password:
[root@backup ~]#</pre>

<p>Запускаем сервис <i>postresql</i>:</p>

<pre>[root@backup ~]# <b>systemctl enable postgresql-14 --now</b>
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql-14.service to /usr/lib/systemd/system/postgresql-14.service.
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>systemctl status postgresql-14</b>
● postgresql-14.service - PostgreSQL 14 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-14.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-11-09 09:31:52 UTC; 39s ago
     Docs: https://www.postgresql.org/docs/14/static/
  Process: 22483 ExecStartPre=/usr/pgsql-14/bin/postgresql-14-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 22488 (postmaster)
   CGroup: /system.slice/postgresql-14.service
           ├─22488 /usr/pgsql-14/bin/postmaster -D /var/lib/pgsql/14/data/
           ├─22490 postgres: logger
           ├─22491 postgres: startup recovering 000000010000000000000005
           ├─22492 postgres: checkpointer
           ├─22493 postgres: background writer
           ├─22494 postgres: stats collector
           └─22495 postgres: walreceiver streaming 0/5000060

Nov 09 09:31:50 backup systemd[1]: Starting PostgreSQL 14 database server...
Nov 09 09:31:50 backup postmaster[22488]: 2022-11-09 09:31:50.370 UTC [22488...s
Nov 09 09:31:50 backup postmaster[22488]: 2022-11-09 09:31:50.370 UTC [22488....
Nov 09 09:31:52 backup systemd[1]: Started PostgreSQL 14 database server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@backup ~]#</pre>



<pre>[root@backup ~]# <b>sudo -u postgres pg_dump -h 192.168.50.10 -d backup --create > /backup/backup.sql</b>
could not change directory to "/root": Permission denied
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>ls -l /backup/backup.sql</b>
-rw-r--r--. 1 root root 1852 Nov  9 19:42 /backup/backup.sql
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>sudo -u postgres pg_dump -h 192.168.50.10 -d backup --create | gzip > /backup/backup.gz</b>
could not change directory to "/root": Permission denied
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>ls -l /backup/backup.gz</b>
-rw-r--r--. 1 root root 615 Nov  9 19:45 /backup/backup.gz
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>sudo -u postgres pg_dump -h 192.168.50.10 -d backup -Fc > /backup/custom.gz</b>
could not change directory to "/root": Permission denied
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>ls -l /backup/custom.gz</b>
-rw-r--r--. 1 root root 1764 Nov  9 19:49 /backup/custom.gz
[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>cat /backup/backup.sql</b>
--
-- PostgreSQL database dump
--

-- Dumped from database version 14.5
-- Dumped by pg_dump version 14.5

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: backup; Type: DATABASE; Schema: -; Owner: postgres
--

CREATE DATABASE backup WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE = 'en_US.UTF-8';


ALTER DATABASE backup OWNER TO postgres;

\connect backup

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: fruits; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.fruits (
    id integer,
    name text,
    count integer
);


ALTER TABLE public.fruits OWNER TO postgres;

--
-- Name: fruits2; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.fruits2 (
    id integer,
    name text,
    count integer
);


ALTER TABLE public.fruits2 OWNER TO postgres;

--
-- Data for Name: fruits; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.fruits (id, name, count) FROM stdin;
1	apple	7
2	pear	3
3	banana	2
\.


--
-- Data for Name: fruits2; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.fruits2 (id, name, count) FROM stdin;
1	apple	7
2	pear	3
3	banana	2
\.


--
-- PostgreSQL database dump complete
--

[root@backup ~]#</pre>

<pre>[root@backup ~]# <b>cat /backup/custom.gz</b>
PGDMP1	
zbackup14.514.s
                  0ENCODINENCODINGSET client_encoding = 'UTF8';
falset
                     00
STDSTRINGS
STDSTRINGS(SET standard_conforming_strings = 'on';
falseu
                     00
SEARCHPATH
SEARCHPATH8SELECT pg_catalog.set_config('search_path', '', false);
falsev
                     126216390backuDATABASE[CREATE DATABASE backup WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE = 'en_US.UTF-8';
DROP DATABASE backup;
postgresfalse�125916391fruitsTABLEQCREATE TABLE public.fruits (
    id integer,
    name text,
    count integer
);
DROP TABLE public.fruits;
publicheapostgresfalse�125916396fruits2TABLERCREATE TABLE public.fruits2 (
    id integer,
    name text,
    count integer
);

ROP TABLE public.fruits2;
publicheapostgresfalseo
                              016391fruits
TABLE DATA1COPY public.fruits (id, name, count) FROM stdin;
publicpostgresfalse209rp
                                 016396fruits2
TABLE DATA2COPY public.fruits2 (id, name, count) FROM stdin;
publicpostgresfalse210�o
                                  )x�3�L,(�I�4�2�,HM,�4�2�LJ�BN#�=...�Jp
                                                                           )x�3�L,(�I�4�2�,HM,�4�2�LJ�BN#�=...�J[root@backup ~]#</pre>

<p>Теперь будем пытаться восстанавливать эти базы.</p>

<p>backup.sql</p>

<pre>[root@master ~]# <b>sudo -u postgres psql -c "DROP DATABASE backup;"</b>
could not change directory to "/root": Permission denied
DROP DATABASE
[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql < /backup/backup.sql</b>
could not change directory to "/root": Permission denied
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
CREATE DATABASE
ALTER DATABASE
You are now connected to database "backup" as user "postgres".
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
CREATE TABLE
ALTER TABLE
COPY 3
COPY 3
[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql -d backup -c "SELECT * FROM fruits;"</b>
could not change directory to "/root": Permission denied
 id |  name  | count 
----+--------+-------
  1 | apple  |     7
  2 | pear   |     3
  3 | banana |     2
(3 rows)

[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql -c "DROP DATABASE backup;"</b>
could not change directory to "/root": Permission denied
DROP DATABASE
[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql -c "CREATE DATABASE backup;"</b>
could not change directory to "/root": Permission denied
CREATE DATABASE
[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres pg_restore /backup/custom.gz -d backup</b>
could not change directory to "/root": Permission denied
[root@master ~]#</pre>

<pre>[root@master ~]# <b>sudo -u postgres psql -d backup -c "SELECT * FROM fruits;"</b>
could not change directory to "/root": Permission denied
 id |  name  | count 
----+--------+-------
  1 | apple  |     7
  2 | pear   |     3
  3 | banana |     2
(3 rows)

[root@master ~]#</pre>

### Создать systemd:
### - pg_backup.service
### - pg_backup.timer
### - pg_backup.sh

vi pg_backup.sh
Вариант 1. Запуск от пользователя root; одна база.

#!/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

PGPASSWORD=password
export PGPASSWORD
pathB=/backup
dbUser=dbuser
database=db

find $pathB \( -name "*-1[^5].*" -o -name "*-[023]?.*" \) -ctime +61 -delete
pg_dump -U $dbUser $database | gzip > $pathB/pgsql_$(date "+%Y-%m-%d").sql.gz

unset PGPASSWORD

* где password — пароль для подключения к postgresql; /backup — каталог, в котором будут храниться резервные копии; dbuser — имя учетной записи для подключения к БУБД; pathB — путь до каталога, где будут храниться резервные копии.
* данный скрипт сначала удалит все резервные копии, старше 61 дня, но оставит от 15-о числа как длительный архив. После при помощи утилиты pg_dump будет выполнено подключение и резервирование базы db. Пароль экспортируется в системную переменную на момент выполнения задачи.

Для запуска резервного копирования по расписанию, сохраняем скрипт в файл, например, /scripts/postgresql_dump.sh и создаем задание в планировщике:

crontab -e

3 0 * * * /scripts/postgresql_dump.sh

* наш скрипт будет запускаться каждый день в 03:00.

Вариант 2. Запуск от пользователя postgres; все базы.

#!/bin/bash
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

pathB=/backup/postgres

find $pathB \( -name "*-1[^5].*" -o -name "*-[023]?.*" \) -ctime +61 -delete

for dbname in `echo "SELECT datname FROM pg_database;" | psql | tail -n +3 | head -n -2 | egrep -v 'template0|template1|postgres'`; do
    pg_dump $dbname | gzip > $pathB/$dbname-$(date "+%Y-%m-%d").sql.gz
done;

* где /backup — каталог, в котором будут храниться резервные копии; pathB — путь до каталога, где будут храниться резервные копии.
* данный скрипт сначала удалит все резервные копии, старше 61 дня, но оставит от 15-о числа как длительный архив. После найдет все созданные в СУБД базы, кроме служебных и при помощи утилиты pg_dump будет выполнено резервирование каждой найденной базы. Пароль нам не нужен, так как по умолчанию, пользователь postgres имеет возможность подключаться к базе без пароля.

Необходимо убедиться, что у пользователя postgre будет разрешение на запись в каталог назначения, в нашем примере, /backup/postgres.

Зададим в качестве владельца файла, пользователя postgres:

chown postgres:postgres /scripts/postgresql_dump.sh

Для запуска резервного копирования по расписанию, сохраняем скрипт в файл, например, /scripts/postgresql_dump.sh и создаем задание в планировщике:

crontab -e -u postgres

* мы откроем на редактирование cron для пользователя postgres.

3 0 * * * /scripts/postgresql_dump.sh

* наш скрипт будет запускаться каждый день в 03:00.

Права и запуск

Разрешаем запуск скрипта, как исполняемого файла:

chmod +x /scripts/postgresql_dump.sh

Единоразово можно запустить задание на выполнение резервной копии:

/scripts/postgresql_dump.sh

... или от пользователя postgres:

su - postgres -c "/scripts/postgresql_dump.sh"

На удаленном сервере
Если сервер баз данных находится на другом сервере, просто добавляем опцию -h:

pg_dump -h 192.168.0.15 users > /tmp/users.dump

* необходимо убедиться, что сама СУБД разрешает удаленное подключение. Подробнее читайте инструкцию Как настроить удаленное подключение к PostgreSQL.







