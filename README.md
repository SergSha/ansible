<h3>### Ansible ###</h3>

<p>Первые шаги с Ansible</p>

<h4>Описание домашнего задания</h4>
<p>Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:</p>
<ul>
  <li>необходимо использовать модуль yum/apt;</li>
  <li>конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными;</li>
  <li>после установки nginx должен быть в режиме enabled в systemd;</li>
  <li>должен быть использован notify для старта nginx после установки;</li>
  <li>сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible.</li>
</ul>

<h4>Критерии оценки:</h4>
<p>Статус "Принято" ставится, если:</p>
<ul>
  <li>предоставлен Vagrantfile и готовый playbook/роль (инструкция по запуску стенда, если посчитаете необходимым);</li>
  <li>после запуска стенда nginx доступен на порту 8080;</li>
  <li>при написании playbook/роли соблюдены перечисленные в задании условия.</li>
</ul>

<h4>Установка Ansible</h4>

<p>Убедимся, что у нас установлена нужная (2.6) или более новая версия:</p>

<pre>[user@localhost ~]$ python -V
Python 2.7.5
[user@localhost ~]$</pre>

<p>Была произведена по инструкции установка Ansible и убедимся, что установлена требуемая (2.4) или более новая версия:</p>

<pre>[user@localhost ~]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
[user@localhost ~]$</pre>

<h4>Настройка Ansible</h4>

<p>● Для управления хостами Ansible использует SSH соединение. Поэтому перед стартом необходимо убедиться что у Вас есть доступ до управляемых хостов.<br />
● Также на управляемых хостах должен быть установлен Python 2.X</p>

<h4>Подготовка окружения</h4>

<p>В домашней директории создадим каталог ansible:</p>

<pre>[user@localhost otus]$ mkdir ./ansible
[user@localhost otus]$</pre>

<p>Перейдём в этот каталог:</p>

<pre>[user@localhost otus]$ cd ./ansible/
[user@localhost ansible]$</pre>

<p>В этом каталоге создадим файл Vagrantfile:</p>

<pre>[user@localhost ansible]$ vi ./Vagrantfile</pre>

<p>Заполним его содержимым из https://gist.github.com/lalbrekht/f811ce9a921570b1d95e07a7dbebeb1e:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :nginx => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.150'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "200"]
          end
          
          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL

      end
  end
end
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost ansible]$ vagrant up</pre>

<p>Смотрим статус запущенной ВМ:</p>

<pre>[user@localhost ansible]$ vagrant status
Current machine states:

nginx                     running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.
[user@localhost ansible]$</pre>

<p>Пробуем подключиться к ней с помощью SSH:</p>

<pre>[user@localhost ansible]$ vagrant ssh
[vagrant@nginx ~]$</pre>

<p>Как видим, подключение к этой ВМ по SSH прошло успешно.</p>

<p>Для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью команды vagrant ssh-config:</p>

<pre>[user@localhost ansible]$ vagrant ssh-config
Host nginx  <----- имя хоста
  HostName 127.0.0.1  <----- IP адрес
  User vagrant  <---- имя пользователя, под которым подключаемся
  Port 2222 <---- порт, который проброшен на 127.0.0.1
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/user/otus/ansible/.vagrant/machines/nginx/virtualbox/private_key <----- путь до приватного ключа
  IdentitiesOnly yes
  LogLevel FATAL

[user@localhost ansible]$</pre>

<p>Используя эти параметры создадим свой первый inventory файл. Предварительно создадим директории staging/hosts:</p>

<pre>[user@localhost ansible]$ mkdir -p ./staging/hosts
[user@localhost ansible]$</pre>

<pre>[user@localhost ansible]$ vi ./staging/hosts/inventory</pre>

<pre>[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key</pre>

<p>И наконец убедимся, что Ansible может управлять нашим хостом. Сделаем это с помощью команды:</p>

<pre>[user@localhost ansible]$ ansible nginx -i staging/hosts -m ping
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ECDSA key fingerprint is SHA256:5IqtRy+Gdd69kALUgBiagwNvU9ms2kqhwLU34BXM/tw.
ECDSA key fingerprint is MD5:af:1f:06:9d:21:12:c5:57:c5:55:6e:fc:02:d2:d2:5a.
Are you sure you want to continue connecting (yes/no)? yes
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
[user@localhost ansible]$</pre>

<p>Чтобы нам не приходилось каждый раз явно указывать наш inventory файл и вписывать в него много информация, создадим в текущей директории ansible.cfg файл, прописав конфигурацию в нем:</p>

<pre>[user@localhost ansible]$ vi ansible.cfg</pre>

<pre>[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False</pre>

<p>Теперь из inventory убираем информацию о пользователе:</p>

<pre>[user@localhost ansible]$ vi ./staging/hosts/inventory</pre>

<pre>[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key</pre>

<p>Еще раз убедимся, что управляемый хост доступен, только теперь без явного указания inventory файла:</p>

<pre>[user@localhost ansible]$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
[user@localhost ansible]$</pre>

<p>Как видим, команда сработала успешно.</p>

<p>Теперь, когда мы убедились, что у нас все подготовлено - установлен Ansible, поднят хост для теста и Ansible имеет к нему доступ, мы можем конфигурировать наш хост.<br />
Для начала воспользуемся Ad-Hoc командами и выполним некоторые удаленные команды на нашем хосте.</p>

<p>Посмотрим какое ядро установлено на хосте:</p>

<pre>[user@localhost ansible]$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64
[user@localhost ansible]$</pre>

<p>Проверим статус сервиса firewalld:</p>

<pre>[user@localhost ansible]$ ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "name": "firewalld", 
    "status": {
        "ActiveEnterTimestampMonotonic": "0", 
        "ActiveExitTimestampMonotonic": "0", 
        "ActiveState": "inactive", <----- не активен
        ...
    }
}
[user@localhost ansible]$</pre>

<p>Установим пакет epel-release на наш хост:</p>

<pre>[user@localhost ansible]$ ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "epel-release"
        ]
    }, 
    ...
[user@localhost ansible]$</pre>

<p>Напишем простой Playbook, который будет делать одно из действий, например: установку пакета epel-release. Создадим файл epel.yml со следующим содержимым:</p>

<pre>[user@localhost ansible]$ vi epel.yml</pre>

<pre>---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
    - name: Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present</pre>

<p>После чего запустим выполнение Playbook:</p>

<pre>[user@localhost ansible]$ ansible-playbook epel.yml 

PLAY [Install EPEL Repo] **************************************************************

TASK [Gathering Facts] ****************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ***********************************
ok: [nginx]

PLAY RECAP ****************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$</pre>

<p>Теперь выполним команду удаления epel-release:</p>

<pre>[user@localhost ansible]$ ansible nginx -m yum -a "name=epel-release state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "epel-release"
        ]
    }, 
...
[user@localhost ansible]$</pre>

<p>Снова запустим выполнение Playbook:</p>

<pre>[user@localhost ansible]$ ansible-playbook epel.yml 

PLAY [Install EPEL Repo] **************************************************************

TASK [Gathering Facts] ****************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ***********************************
changed: [nginx]

PLAY RECAP ****************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$</pre>

<p>В этот раз говорит о том, что выполнении команды произошло изменение (changed: nginx), количество изменений равно 1 (changed=1).</p>

<p>Теперь собственно приступим к выполнению домашнего задания и написания Playbook-а для установки NGINX.</p>

<p>За основу возьмем уже созданный нами файл epel.yml и переименуем его в nginx.yml:</p>

<pre>[user@localhost ansible]$ mv ./epel.yml ./nginx.yml 
[user@localhost ansible]$</pre>

<p>И первым делом добавим в него установку пакета NGINX:</p>

<pre>    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages</pre>

<p>Целиком файл будет выглядеть так:</p>

<pre>[user@localhost ansible]$ vi ./nginx.yml</pre>

<pre>---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
</pre>

<p>Обратите внимание - добавили Tags. Теперь можно вывести в консоль список тегов и выполнить, например, только установку NGINX. В нашем случае так, например, можно осуществлять его обновление.<br />
Выведем в консоль все теги:</p>

<pre>[user@localhost ansible]$ ansible-playbook nginx.yml --list-tags

playbook: nginx.yml

  play #1 (nginx): NGINX | Install and configure NGINX	TAGS: []
      TASK TAGS: [epel-package, nginx-package, packages]
[user@localhost ansible]$</pre>

<p>Запустим только установку NGINX:</p>

<pre>[user@localhost ansible]$ ansible-playbook nginx.yml -t nginx-package

PLAY [NGINX | Install and configure NGINX] ********************************************

TASK [Gathering Facts] ****************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX package from EPEL Repo] ***********************************
changed: [nginx]

PLAY RECAP ****************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$</pre>

<p>Далее добавим шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост:</p>

<pre>    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx-configuration</pre>

<p>Сразу же пропишем в Playbook необходимую нам переменную. Нам нужно чтобы NGINX слушал на порту 8080:</p>

<pre>- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080</pre>
    
<p>В итоге на данном этапе Playbook будет выглядеть так:</p>

<pre>[user@localhost ansible]$ vi ./nginx.yml</pre>

<pre>---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx-configuration
</pre>









