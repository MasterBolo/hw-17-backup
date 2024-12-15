
# Домашняя работа: Резервное копирование


	Цель работы: Научиться настраивать резервное копирование с помощью утилиты Borg.

	Что нужно сделать?

	1. Настроить стенд Vagrant с двумя виртуальными машинами: backupsrv и client. 
    2. Настроить удаленный бэкап каталога /etc c сервера client при помощи borgbackup.
	
	Резервные копии должны соответствовать следующим критериям:
    - директория для резервных копий /var/backup.
    - репозиторий для резервных копий должен быть зашифрован ключом или паролем - на усмотрение студента;
    - имя бэкапа должно содержать информацию о времени снятия бекапа;
    - глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. 
    - резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
    - написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на усмотрение студента;
    - настроено логирование процесса бекапа. 

	
	3. Все настройки сделать с помощью Ansible и добавить запуск Ansible playbook из Vagrantfile.

  # Выполнение

## Настроить стенд Vagrant с двумя виртуальными машинами: backupsrv и client.

	Создаем в домашней директории Vagrantfile, собираю стенд на основе ubuntu-22.04.
	Дальнейший разбор настроек будем проводить на примере playbook файлов. Все необходимые файлы конфигурации для настройки
кладём в созданную директорию files. 

## Настраиваем сервер backupsrv
 
	Создаём фаил backupsrv.yml и дополняем его tasks(задачами).

	Для правильной работы с логами необходимо настроить время. Мой часовой пояс - Asia/Yekaterinburg. 

Cоздадим задачу настройки тайм-зоны:
```
tasks: 
    - name: set timezone to Asia/Yekaterinburg
      timezone:
        name: Asia/Yekaterinburg

````
	Создаём блочное устройство размером 2Гб, форматируем, создаём раздел 
для бэкапирования, монтируем диск:
```
tasks: 

    - name: create partition
      parted:
        device: /dev/sdb
        number: 1
        state: present
        part_end: 2GB

    - name: format the ext4 filesystem
      filesystem:
        fstype: ext4
        dev: /dev/sdb1

    - name: Create directory backup
      file:
        path: /var/backup
        state: directory
        mode: '0755'

    - name: mount disk
      mount:
        path: /var/backup
        src: /dev/sdb1
        fstype: ext4
        state: mounted
```
	Обновлем кэш для apt, инсталируем borgbackup :
````
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: borgbackup | Install borgbackup
      apt:
        name: borgbackup
        state: latest
      tags:
        - borgbackup-package
````
	Создадим пользователя borg, владельца для директории /var/backup, директорию .ssh :

```
- name: Create user borg
      user:
        name: borg
        shell: /bin/bash
        create_home: yes

    - name: сreate owner for folder
      file:
        path=/var/backup
        mode=0755
        owner=borg
        state=directory

    - name: сreate owner for folder
      file:
        path=/home/borg/.ssh
        mode=0700
        owner=borg
        state=directory

```
	Создаём фаил "authorized_keys" c владельцем borg, копируем в него созданный заранее публичный ключ сервера client :
````
- name: create file
      file:
        path: "/home/borg/.ssh/authorized_keys"
        state: touch
        owner: borg
        mode: 0600

    - name: copy public key
      ansible.builtin.template:
        src: files/id_rsa.pub
        dest: /tmp/backupsrv_id_rsa.pub

    - name: get serverBackup public key
      ansible.builtin.command: cat /tmp/backupsrv_id_rsa.pub
      register: serverPublicKey

    - name: add server public key to authorized_keys
      ansible.builtin.lineinfile:
        path: /home/borg/.ssh/authorized_keys
        line: '{{serverPublicKey.stdout}}'
````

# Настраиваем сервер client

	Создаём фаил client.yml и дополняем его tasks(задачами).

	Cоздадим задачу настройки тайм-зоны:
```
tasks: 
    - name: set timezone to Asia/Yekaterinburg
      timezone:
        name: Asia/Yekaterinburg

````	
	Обновлем кэш для apt, инсталируем borgbackup :
````
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: borgbackup | Install borgbackup
      apt:
        name: borgbackup
        state: latest
      tags:
        - borgbackup-package
````	
	Копируем на сервер созданные ранее публичный и приватный ключи:
````
    - name: copy public key
      ansible.builtin.template:
        src: files/id_rsa.pub
        dest: /root/.ssh/id_rsa.pub

    - name: copy private key
      ansible.builtin.template:
        src: files/id_rsa
        dest: /root/.ssh/id_rsa
        mode: '0600'
````
	Настрайваем конфигурацию для автоматического приёма ключей:
````
    - name: disable StrictHostKeyChecking
      ansible.builtin.lineinfile:
        path: /etc/ssh/ssh_config
        regexp: 'StrictHostKeyChecking'
        line: StrictHostKeyChecking no
````
	Инициализируем репозиторий borg на backup сервере с client сервера::
````
     - name: initialize borg repository
       ansible.builtin.expect: 
         command: borg init --encryption=repokey borg@192.168.56.10:/var/backup/otus
         responses:
           "Enter new passphrase:":
             - "120806"
           "Enter same passphrase again:":
             - "120806"
           "Do you want your passphrase to be displayed for verification?":
             - "N"
````
	Создаём файлы конфигураций для сервиса и таймера borg, копируя файлы с листингом предоставленным в методичке :

````
    - name: create systemd service for backup
      ansible.builtin.template:
       src: files/borg-backup.service
       dest: /etc/systemd/system/borg-backup.service

    - name: create systemd timer for backup
      ansible.builtin.template:
       src: files/borg-backup.timer
       dest: /etc/systemd/system/borg-backup.timer
````
	 Включаем и стартуем созданные ранее сервисы:

````
    - name: enable borg-backup.timer
      ansible.builtin.service:
       name: borg-backup.timer
       enabled: yes
       state: started

    - name: start borg-backup.service
      ansible.builtin.service:
       name: borg-backup.service
       state: started  

````

Работа завершена.





