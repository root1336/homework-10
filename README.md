# Автоматизация администрирования. Ansible  

## Первые шаги с Ansible  

1. На данном этапе делаю первые попытки разобраться с Ansible и повторить последовательность действий методички. Для этого запускаю стенд https://gitlab.com/otus_linux/09-1-ansible  
	```
	vagrant up
	```
2. Подключаемся к машине Ansible:  
	```
	user@linux1:~/linux/homework-10$ vagrant ssh ansible
	Last login: Sun Mar 22 17:33:56 2020 from 10.0.2.2
	[vagrant@ansible ~]$
	```
3. Теперь убедимся, что с машины Ansible есть доступ на машину web по ssh:  
	```
	[vagrant@ansible ~]$ ssh web
	Last login: Sun Mar 22 17:34:35 2020 from 192.168.11.150
	[vagrant@web ~]$ exit
	logout
	Connection to web closed.
	[vagrant@ansible ~]$
	```
4. На данном шаге проверим, что Ansible установлен и работает:  
	```
	[vagrant@ansible ~]$ ansible --version
	ansible 2.9.6
	  config file = /etc/ansible/ansible.cfg
	  configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
	  ansible python module location = /usr/lib/python2.7/site-packages/ansible
	  executable location = /usr/bin/ansible
	  python version = 2.7.5 (default, Apr  9 2019, 14:30:50) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
	```

5. Теперь выходим из виртуалки и узнаем параметры для подключения к хосту web с помощью команды vagrant ssh-config. Как оказалось в дальнейшем, здесь скрывается один ньюанс, описанный позднее.  
	```
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
	```
6. Далее несколько пунктов выполняю на хостовой машине с целью изучения материала и с дальнейшей мыслью перекинуть затем результаты работы на виртуалку ansible. Создаю в папке inventories/staging файл hosts с описанием виртуалки web, на которой будем разворачивать nginx. В описании применяем параметры, полученные ранее командой vagrant ssh-config:  
	```
	user@linux1:~/linux/homework-10$ cat  inventories/staging/hosts
	[proxy]
	web ansible_host=127.0.0.1 ansible_port=2200 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/web/virtualbox/private_key
	```
7. Теперь убедимся, что Ansible может подключиться по ssh и управлять нашей машиной web.  
	```
	user@linux1:~/linux/homework-10$ ansible web -i inventories/staging/hosts -m ping
	web | SUCCESS => {
	    "ansible_facts": {
		"discovered_interpreter_python": "/usr/bin/python"
	    }, 
	    "changed": false, 
	    "ping": "pong"
	}
	```
8. Теперь создадим плейбук к машине web. Для этого создаем файл nginx.yml в папке playbooks.
В файле указываем название управляемой машины, задаем флаг переключения в режим рута, описываем переменные - используемый порт для nginx и путь к репозиторию, а затем прописываем последовательность задач по скачиванию, установке и запуску nginx. Файл получается следующего содержания:  
	```
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
	```
9. Настало время запустить на исполнение плейбук и убедиться, что в результате выполнения nginx установлен и запущен:  
	```
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
	```
	```
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
	```
10. Следущим шагом изменим порт, на котором работает nginx. Для этого возьмем конфиг-файл установленного nginx, удалим закоммментированные строки и укажем требуемый порт 8080 (через имеющуюся переменную) вместо стандартного порта 80. Данный файл разместим в папке шаблонов.  
	```
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
	```
11. А также создадим в папке шаблонов новый файл приветствия nginx:  
	```
	user@linux1:~/linux/homework-10$ cat templates/index.html.j2
	<h> My hostname is {{ ansible_hostname }} </h>
	```
12. Напоследок добавим в плейбук nginx.yml две новые задачи по подмене файла конфигурации и стартовой страницы и обработчик для перезапуска nginx. Обработчик будет вызываться после подмены файлов для перезапуска nginx (в данном конкретном случае этот способ не рекомендуется, т.к запуск обработчика не будет вызван если при выполнении плейбука не будет изменен файл конфигурации - здесь это сделано только для демонстрации).  
	```
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
	```
13. Запускаем на исполнение плейбук и убеждаемся, что nginx теперь работает на другом порту - 8080 и выдает новую стартовую страницу.
	```
	user@linux1:~/linux/homework-10$ curl 192.168.11.151:8080
	<h> My hostname is web </h>user@linux1:~/linux/homework-10$ 
	```

## Выполнение основного ДЗ 

1. Создадим в основной папке файл конфигурации, в котором укажем инвентори и пользователя
	```
	user@linux1:~/linux/homework-10$ cat ansible.cfg
	[defaults]
	inventory = inventories/staging/
	remote_user = vagrant
	```
2. Вместо файла hosts создадим файл all.yml и пропишем в нем параметры машины web (создадим разные типы серверов и отнесем машину web к типу прокси-серверов)
user@linux1:~/linux/homework-10$ cat inventories/staging/all.yml
	```
	all:
	  children:
	    proxy:
	    db:
	    app:
	  vars:
	    ansible_user: 'vagrant'

	proxy:
	  hosts:
	    web:
	      ansible_host: 192.168.11.151
	      ansible_port: 22
	      ansible_ssh_private_key_file: /home/vagrant/.ssh/id_rsa
	```
3. Для демонстрации работы плейбука изменим взятый со стенда Vagrantfile - прописываем копирование наших созданных папок и файла конфигурации в папку ~/ansible машины ansible. Осталось только поднять машины и проверить отработку плейбука ансиблом, но не тут-то было, именно здесь и скрылась засада, с которой пришлось порядочно поразбираться, штудировать разные мануалы и перерывать все возможные конфиги - ansible никак не хотел соединяться с машиной web ну и пинг, конечно, не проходил. Хотя в итоге проблема оказалась пустяковой и лежала на поверхности. Затык оказался в том, что вагрант пробрасывает порты к виртулкам через адрес localhost, и все изучение ansible началось именно с параметров подключения к машине web через localhost и проброшенный порт 2200, а когда мы перенесли весь код на машину ansible демонстарционного стенда, то там, разумеется никаких настроек проброса порта не было. В итоге переписал настройки подключения на указанный в вагранте адрес виртуалки 192.168.11.151 и стандартного порта 22, после этого все завелось (в описании выполнения ДЗ уже указал нормальный адрес и порт). По результатам произведенных действий осталась пара вопросов: 1) На практике какой подход предпочтительнее? Указывать при работе с ansible реальные адреса поднятых нами виртуалок, или все-таки использовать адрес физического сервера и проброшенные порты? У меня ощущение, что при массовом выполнении однотипных задач на поднятых серверах могут быть различные нюансы и это может иметь значение. 2) Когда разбирался проблемой подключения, что-то не понял как посмотреть настройки проброшенных портов с помощью iptables. Быстрее разобрался с помощью других команд.

4. Поднимаем вагрантом обе наши виртаулки после чего подключаемся к машине ansible, затем переходим в папку ansible и запускаем созданный ранее плейбук 
	```
	user@linux1:~/linux/homework-10$ vagrant ssh ansible
	Last login: Tue Mar 31 16:49:17 2020 from 10.0.2.2
	[vagrant@ansible ~]$ cd ansible
	[vagrant@ansible ansible]$ ansible-playbook playbooks/nginx.yml

	PLAY [web] *********************************************************************

	TASK [Gathering Facts] *********************************************************
	The authenticity of host '192.168.11.151 (192.168.11.151)' can't be established.
	ECDSA key fingerprint is SHA256:Vf//4KHEF0k2nZmuGv/o+Z3RUpqnSVdTDvnNkqNmGzo.
	ECDSA key fingerprint is MD5:1b:55:85:e9:6b:f5:8a:93:6b:f8:f0:a9:f2:15:4a:8c.
	Are you sure you want to continue connecting (yes/no)? yes
	ok: [web]

	TASK [Create file for NGINX repo] **********************************************
	changed: [web]

	TASK [Add official NGINX repo] *************************************************
	changed: [web]

	TASK [Install NGINX] ***********************************************************
	changed: [web]

	TASK [Start NGINX server] ******************************************************
	changed: [web]

	TASK [Copy index.html] *********************************************************
	changed: [web]

	TASK [Copy default.conf] *******************************************************
	changed: [web]

	RUNNING HANDLER [reload nginx] *************************************************
	changed: [web]

	PLAY RECAP *********************************************************************
	web                        : ok=8    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
	```
5. Проверяем, что nginx поднят и работает на указанном нами порту 8080
	```
	[vagrant@ansible ansible]$ curl web:8080
	<h> My hostname is web </h>[vagrant@ansible ansible]$ 
	```

