# Описание

Стенд для практики на уроке "Автоматизация с помощью Ansible"

## Quick Start

Для запуска стенда достаточно дать:

```
vagrant up
```

Поднимется две машины:

* `ansible` - с уже установленным пакетом `ansible` и ключом для доступа к хосту `web`
* `web` - управляемый хост с которым будем экспериментировать

Задание находится по ссылке: https://docs.google.com/document/d/1xaQZVyzw37kbZjk43sXo3xHV74ukGeQpFeyLwIwTNwQ/edit?usp=sharing

1. Запустить стенд - vagrant up

2. Зайти на Ansible host:
user@linux1:~/linux/homework-10$ vagrant ssh ansible
Last login: Sun Mar 22 17:33:56 2020 from 10.0.2.2
[vagrant@ansible ~]$

3. Убедиться что есть доступ на хост web по ssh:
[vagrant@ansible ~]$ ssh web
Last login: Sun Mar 22 17:34:35 2020 from 192.168.11.150
[vagrant@web ~]$ exit
logout
Connection to web closed.
[vagrant@ansible ~]$

4. Убедиться что Ansible установлен и работает: 
[vagrant@ansible ~]$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  9 2019, 14:30:50) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]


5. Узнаем параметры для подключения к хосту web с помощью команды vagrant ssh-config 
user@linux1:~/linux/homework-10$ vagrant ssh-config
Host ansible
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/user/linux/homework-10/.vagrant/machines/ansible/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

Host web
  HostName 127.0.0.1
  User vagrant
  Port 2200
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/user/linux/homework-10/.vagrant/machines/web/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

6. Создадим в папке inventories/staging файл hosts со следующим содержимым 
user@linux1:~/linux/homework-10$ cat  inventories/staging/hosts
[proxy]
web ansible_host=127.0.0.1 ansible_port=2200 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/web/virtualbox/private_key

7. И наконец убедимся, что Ansible может управлять нашим хостом. 
user@linux1:~/linux/homework-10$ ansible web -i inventories/staging/hosts -m ping
web | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}

