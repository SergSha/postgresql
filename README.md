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
#      if boxconfig[:vm_name] == "replica"
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

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost postgresql]$</pre>

<h4>Сервер master</h4>

<p>Подключаемся по ssh к серверу master и зайдём с правами root:</p>

<pre>[user@localhost postgresql]$ <b>vagrant ssh master</b>
[vagrant@master ~]$ <b>sudo -i</b>
[root@master ~]#</pre>

<p>Подключаем репозиторий ... последней версии:</p>



























