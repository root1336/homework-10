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

8. Создаем файл nginx.yml в папке playbooks следющего содержания
user@linux1:~/linux/homework-10$ cat playbooks/nginx.yml
- hosts: web
  become: true
  vars:
    nginx_port: 8080
    nginx_repo_path: /etc/yum.repos.d/nginx.repo

  tasks:

    - name: 'Create file for NGINX repo'
      file:
        path: "{{ nginx_repo_path }}"
        state: touch

    - name: 'Add official NGINX repo'
      blockinfile:
        path: "{{ nginx_repo_path }}"
        block: |
          [nginx-stable]
          name=nginx stable repo
          baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
          gpgcheck=1
          enabled=1
          gpgkey=https://nginx.org/keys/nginx_signing.key
          module_hotfixes=true 

    - name: 'Install NGINX'
      yum:
        name: nginx
        state: present

    - name: 'Start NGINX server'
      systemd:
        name: nginx
        state: started
        enabled: true

9. Запустим на исполнение плейбук и убедимся, что nginx установлен и запущен
user@linux1:~/linux/homework-10$ ansible-playbook -i inventories/staging playbooks/nginx.yml

PLAY [web] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [web]

TASK [Create file for NGINX repo] **********************************************
changed: [web]

TASK [Add official NGINX repo] *************************************************
ok: [web]

TASK [Install NGINX] ***********************************************************
ok: [web]

TASK [Start NGINX server] ******************************************************
changed: [web]

PLAY RECAP *********************************************************************
web                        : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

user@linux1:~/linux/homework-10$ curl 192.168.11.151
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

10. Возьмем конфиг-файл установленного nginx, удалим закоммментированные строки и укажем требуемый порт 8080 (через имеющуюся переменную) вместо стандартного порта 80
user@linux1:~/linux/homework-10$ cat templates/default.conf.j2
server {
    listen       {{ nginx_port }};
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

11. Добавим новый файл приветствия nginx:
user@linux1:~/linux/homework-10$ cat templates/index.html.j2
<h> My hostname is {{ ansible_hostname }} </h>

12. Добавим в плейбук nginx.yml две новые задачи по подмене файла конфигурации и стартовой страницы. После подмены файлов будет вызван обработчик для перезапуска nginx (данный способ не рекомендуется, т.к запуск обработчика не будет вызван если при выполнении плейбука не будет изменен файл конфигурации) - здесь это сделано только для демонстрации
    - name: 'Copy index.html'
      template:
        src: ../templates/index.html.j2
        dest: /usr/share/nginx/html/index.html

    - name: 'Copy default.conf'
      template:
        src: ../templates/default.conf.j2
        dest: /etc/nginx/conf.d/default.conf
      notify:
        - reload nginx

  handlers:
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded

13. Запустим на исполнение плейбук и затем убедимся, что nginx теперь работает на другом порту - 8080 и выдает новую стартовую страницу

