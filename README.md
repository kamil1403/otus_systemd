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
• Создал сервис, отрабатывает успешно. Результат см. на скриншоте 🖼️ ["lvm_create"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/lvm_create.png)     


## 🧭 Оглавление

- [📋 Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова](#unit)
- [⚙️ Установить spawn-fcgi и создать unit-файл ](#spawn)
- [✂️ Расширение файловой системы](#resize_max)
- [✂️ Уменьшение файловой системы](#resize_min)
- [📸 Работа со снепшотами](#snapshot)
- [📋 Монтирование в fstab](#fstab)

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
# Форматирование логического тома
mkfs.ext4 /dev/otus/test
# Монтирование логического тома
mkdir /lvm
mount /dev/otus/test /lvm
```

---

<a id="resize_max"></a>
## ✂️ Расширение файловой системы

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

<a id="resize_min"></a>
## ✂️ Уменьшение файловой системы

```bash
# Отключение логического тома   
umount /lvm   
# Проверка файловой системы   
e2fsck -f /dev/otus/test   
# Уменьшение файловой системы   
resize2fs /dev/otus/test 4G   
# Уменьшение логического тома   
lvreduce -L 4G /dev/otus/test   
# Монтирование логического тома
mount /dev/otus/test /lvm
```

---

<a id="snapshot"></a>
## 📸 Работа со снепшотами

```bash
# Копирование файлов в логический том (для теста)
cp -r /var/log/* /lvm
# Создание snapshot   
lvcreate -s -L 1G -n otus_snap /dev/otus/test 
# Создание директории для snapshot
mkdir /snapshot 
# Монтирование snapshot и создание бэкапа  
mount /dev/otus/otus_snap /snapshot  
# Удаление файлов из логического тома
rm -rf /lvm/* 
# Размонтирование логического тома
umount /lvm
umount /snapshot
# Деактирование тома
lvchange -an /dev/otus/test
# Восстановление файлов из snapshot
lvconvert --merge /dev/otus/otus_snap
# Активирование тома
lvchange -ay /dev/otus/test
# Монтирование логического тома
mount /dev/otus/test /lvm
```

---

<a id="fstab"></a>
## 📋 Монтирование в fstab

```bash
# Показывает UUID и тип файловой системы для всех разделов (нужен для настройки /etc/fstab)
blkid
# UUID otus-test
/dev/mapper/otus-test: UUID="3aaf9c58-0608-4fb9-8394-498d3edbc090" BLOCK_SIZE="4096" TYPE="ext4"
# Редактирует файл fstab для автоматического монтирования разделов при загрузке
nano /etc/fstab
# Пример строки для монтирования тома в каталог /lvm
UUID=3aaf9c58-0608-4fb9-8394-498d3edbc090  /lvm ext4 defaults 0 2
```

---
