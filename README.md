<p align="center">
  <img src="https://github.com/kamil1403/otus_systemd/blob/main/screenshots/banner.jpeg" alt="RAID Banner" width="65%">
</p>


<h1 align="center">otus_systemd</h1>
<p align="center">Дата: 30-09-2025<br>Автор: Kamil Ibragimov</p>

## Домашнее задание: systemd - создание unit-файла
Задание:   
• Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default);   
• Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта;   
• Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.   

Результат:   
• Создал сервис, отрабатывает успешно. Результат см. на скриншоте 🖼️ ["watchlog.timer"](https://github.com/kamil1403/otus_systemd/blob/main/screenshots/watchlog.timer.png)     
• Установил spawn-fcgi и создал unit-файл. Результат см. на скриншоте 🖼️ ["spawn-fcgi"](https://github.com/kamil1403/otus_systemd/blob/main/screenshots/spawn.png)     
• Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов. Результат см. на скриншоте 🖼️ ["ng"](https://github.com/kamil1403/otus_systemd/blob/main/screenshots/spawn.png)     

## 🧭 Оглавление

- [📋 Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова](#unit)
- [⚙️ Установить spawn-fcgi и создать unit-файл ](#spawn)
- [🌐 Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов](#resize_max)

---

<a id="unit"></a>
## 📋 Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова

```bash
# ----------------------------- STEP 1 -----------------------------
# Конфиг с переменными:
cd /etc/default/
nano watchlog

# ----------------------------- STEP 2 -----------------------------
# Вставить и сохранить:
# Configuration file for my watchlog service
# Place it to /etc/default
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log

# ----------------------------- STEP 3 -----------------------------
# Лог-файл и тестовые строки (обязательно с ALERT):
cd /var/log/
nano watchlog.log

# ----------------------------- STEP 4 -----------------------------
# Скрипт:
bash
cd /opt
nano watchlog.sh

# ----------------------------- STEP 5 -----------------------------
# Вставить и сохранить:
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi

# ----------------------------- STEP 6 -----------------------------
# Сделать исполняемым:
chmod +x /opt/watchlog.sh

# ----------------------------- STEP 7 -----------------------------
# Юнит сервиса:
cd /etc/systemd/system
nano watchlog.service

# ----------------------------- STEP 8 -----------------------------
# ставить и сохранить:
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG

# ----------------------------- STEP 9 -----------------------------
# Юнит таймера:
nano watchlog.timer

# ----------------------------- STEP 10 ----------------------------
# Вставить и сохранить:
[Unit]
Description=Run watchlog script every 30 second
[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
[Install]
WantedBy=multi-user.target

# ----------------------------- STEP 11 ----------------------------
# Запуск:
systemctl daemon-reload
systemctl start watchlog.service
systemctl start watchlog.timer

# ----------------------------- STEP 12 ----------------------------
# Проверка результата:
tail -n 1000 /var/log/syslog | grep word

# ----------------------------- RESULT -----------------------------
# Готово. Если увидим строку: "I found word, Master!" - значит сервис работает.
```

---

<a id="spawn"></a>
## ⚙️ Установить spawn-fcgi и создать unit-файл 

```bash
# ----------------------------- STEP 1 -----------------------------
# Устанавливаем spawn-fcgi и необходимые для него пакеты:
apt install spawn-fcgi php php-cgi php-cli \ apache2 libapache2-mod-fcgid -y

# ----------------------------- STEP 2 -----------------------------
# Скрипт:
cd /etc/spawn-fcgi
nano fcgi.conf

# ----------------------------- STEP 3 -----------------------------
# Вставить и сохранить:
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

# ----------------------------- STEP 3 -----------------------------
# Юнит сервиса:
cd /etc/systemd/system
nano spawn-fcgi.service

# ----------------------------- STEP 4 -----------------------------
# Вставить и сохранить:
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target

# ----------------------------- STEP 5 ----------------------------
# Запуск:
systemctl daemon-reload
systemctl start spawn-fcgi
systemctl start spawn-fcgi

# ----------------------------- STEP 12 ----------------------------
# Проверка результата:
tail -n 1000 /var/log/syslog | grep word

# ----------------------------- RESULT -----------------------------
# Готово. Если увидим строку: "active (running)" - значит сервис работает.

```

---

<a id="ng"></a>
## 🌐 Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов

```bash
# Инициализация нового PV   
pvcreate /dev/sdc   
vgextend otus /dev/sdc   
# Расширение логического тома   
lvextend -l+80%FREE /dev/otus/test
# Расширение файловой системы
resize2fs /dev/otus/test
```

---
