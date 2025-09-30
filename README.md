# otus_systemd
<p align="center">
  <img src="https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/lvm.jpg" alt="RAID Banner" width="800">
</p>

<h1 align="center">otus_systemd</h1>
<p align="center">Дата: 30-09-2025<br>Автор: Kamil Ibragimov</p>

## Домашнее задание: работа с LVM
Задание:   
• Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default);
• Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020);
• Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно. 

Результат:   
• Создал Physical Volume, Volume Group и Logical Volume. Результат см. на скриншоте 🖼️ ["lvm_create"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/lvm_create.png)     
• Отформатировал том и смонтировал файловую систему в каталог /lvm. Результат см. на скриншоте 🖼️ ["mkfs.ext4"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/mkfs.ext4.png)     
• Расширил файловую систему за счет нового диска на 5Gb. Результат см. на скриншоте 🖼️ ["resize"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/resize.png)   
• Уменьшил файловую систему до 4Gb. Результат см. на скриншоте 🖼️ ["resize_4Gb"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/resize_4G.png)   
• Скопировал файлы в /lvm, создал снепшот, удалил файлы из /lvm, а затем восстановил их из снепшота. Результат см. на скриншоте 🖼️ ["snapshot"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/snapshot.png)  
• Смонтировал том "test" в каталог /lvm. Видно, что после перезагрузки том смонтировался корректно. Результат см. на скриншоте 🖼️ ["fstab"](https://github.com/kamil1403/otus_LVM-1/blob/main/screenshots/fstab.png)  


## 🧭 Оглавление

- [🛠️ Создание Physical Volume, Volume Group и Logical Volume](#pvl)
- [⚙️ Форматирование и монтирование файловой системы](#ext4)
- [✂️ Расширение файловой системы](#resize_max)
- [✂️ Уменьшение файловой системы](#resize_min)
- [📸 Работа со снепшотами](#snapshot)
- [📋 Монтирование в fstab](#fstab)

---

<a id="pvl"></a>
## 🛠️ Создание Physical Volume, Volume Group и Logical Volume

```bash
# LVM (Logical Volume Manager) — это система управления дисками в Linux, позволяющая гибко объединять и перераспределять пространство на физических носителях   
# LVM использует три уровня:
# 1. PV (Physical Volume) — это "сырой" диск или раздел, подготовленный под использование в LVM   
pvcreate /dev/sdb
# 2. VG (Volume Group) — это контейнер, в который объединяются один или несколько PV   
vgcreate otus /dev/sdb
# 3. LV — это "кусок" из группы VG, который работает как обычный раздел, только гибче   
lvcreate -L 10G -n test otus
# Создание логического тома на 80% от доступного места   
lvcreate -l+80%FREE -n test otus   
# Удаление логического тома   
lvremove /dev/otus/test   
# Просмотр и управление LVM   
# Подробно:  
pvdispaly 
vgdisplay   
lvdisplay   
# Кратко:   
pvs   
vgs   
lvs   
```

---

<a id="ext4"></a>
## ⚙️ Форматирование и монтирование файловой системы

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
