**Домашнее задание**

Что нужно сделать?

- Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников


**Выполнение:**

1. Подключаемся к нашей созданной ВМ: 
```
vagrant ssh
```
2. Переходим в root-пользователя: 
```
sudo -i
```
5. Создаём пользователя otusadm и otus: 
```
sudo useradd otusadm && sudo useradd otus
```
8. Создаём пользователям пароли:
``` 
echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
```
Для примера мы указываем одинаковые пароли для пользователя otus и otusadm

5. Создаём группу admin: 
```
sudo groupadd -f admin
```
7. Добавляем пользователей vagrant,root и otusadm в группу admin:
```
usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
```
После создания пользователей, нужно проверить, что они могут подключаться по SSH
```
neon@neon-desktop:/test_vm$ ssh otus@192.168.56.10
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:xIazESKKwTlBTtIOLIj/jCnF+TsCnV5ZodKc2vrSp9E.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.10' (ED25519) to the list of known hosts.
otus@192.168.56.10's password: 
[otus@pam ~]$
```
Проверим, что пользователи root, vagrant и otusadm есть в группе admin:

```
[root@pam vagrant]# cat /etc/group | grep admin
printadmin:x:994:
admin:x:1003:otusadm,root,vagrant
```

8. Создадим файл-скрипт /usr/local/bin/login.sh
```
cat /usr/local/bin/login.sh
#!/bin/bash
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 if getent group admin | grep -qw "$PAM_USER"; then
        exit 0
      else
          exit 1
    fi
  else
    exit 0
fi
[root@pam vagrant]# cat /usr/local/bin/login.sh
#!/bin/bash
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 if getent group admin | grep -qw "$PAM_USER"; then
        exit 0
      else
          exit 1
    fi
  else
    exit 0
fi
```
9. Добавим права на исполнение файла: 
```
chmod +x /usr/local/bin/login.sh
```
10. Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:

```
vi /etc/pam.d/sshd 
...
account    required     pam_exec.so /usr/local/bin/login.sh
...
```
11. Установим дату на выходной день
```
date -s "2023-06-03 10:19:00"
date
Sat Jun  3 10:19:04 UTC 2023
```
12. Проверяем что пользователь otusadm подключается без проблем, в у пользователя otus возникает ошибка:

```
neon@neon-desktop:/test_vm$ ssh otusadm@192.168.56.10
otusadm@192.168.56.10's password: 
Last failed login: Sat Jun  3 10:29:05 UTC 2023 from 192.168.56.1 on ssh:notty
There were 1 failed login attempts since the last successful login.
[otusadm@pam ~]$

neon@neon-desktop:/test_vm$ ssh otus@192.168.56.10
otus@192.168.56.10's password: 
/usr/local/bin/login.sh failed: exit code 1
Connection closed by 192.168.56.10 port 22
```
