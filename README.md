1) Хостовая машина
# Проверяем версию ansible
ansible --version
ansible 2.10.8

# проверим версию
python3 -V
Python 3.10.12

2) Изменил Vagrant файл (ругался на сеть).

3)
# Вывести параметры подключения вагранта
 vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /media/VM/Ansible/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa

4) # проверяем, что у нас подключается по ssh
ssh vagrant@192.168.38.250

5) #создаём инвентори в staging/hosts
mkdir staging
nano ./staging/hosts

[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key

# убеждаемся, что всё работает
ansible nginx -i staging/hosts -m ping

6) # для автоматизации создаём ansible.cfg
nano ansible.cfg

[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False

# редактируем staging/hosts. Убираем из него ansible_user=vagrant, т.к. ansible_user прописан в ansible.cfg

nano ./staging/hosts

[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key

# убеждаемся, что всё работает
ansible nginx -m ping

6) #посмотрим, какое ядро на виртуалке
ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64

# Проверим статус сервиса firewalld
ansible nginx -m systemd -a name=firewalld | grep ActiveState
"ActiveState": "inactive"

# Установим пакет epel-release на наш хост
ansible nginx -m yum -a "name=epel-release state=present" -b
"changed": true

# создадим плейбук epel.yml
nano epel.yml

---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
    - name: Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present

# запустим его
ansible-playbook epel.yml

# создадим nginx.yml
nano nginx.yml

---
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


# просмотрим теги
ansible-playbook nginx.yml --list-tags

# запуск только установки nginx-package
ansible-playbook nginx.yml -t nginx-package

mkdir templates
nano templates/nginx.conf.j2

# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}


# изменим nginx.yml

---
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
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded



# по адресу 192.168.38.250:8080 сайт доступен