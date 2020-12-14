#### LVM
Работа с LVM
на имеющемся образе
/dev/mapper/VolGroup00-LogVol00 38G 738M 37G 2% /

уменьшить том под / до 8G
выделить том под /home
выделить том под /var
/var - сделать в mirror
/home - сделать том для снэпшотов
прописать монтирование в fstab
попробовать с разными опциями и разными файловыми системами ( на выбор)
- сгенерить файлы в /home/
- снять снэпшот
- удалить часть файлов
- восстановится со снэпшота

## Попробуем сделать без перезагрузки 
заводим и цепляемся к виртуалке 
```sh
vagrant up
vagrant ssh 
sudo -s 
```
смотрим имеющееся
```sh
df -T
```
имеем xfs 
```sh
yum install xfsdump lsof -y 
```
бекапим /
```sh
pvcreate /dev/sdb
vgcreate VolGroup_tmp /dev/sdb
lvcreate -l 100%FREE -n LogVol00 VolGroup_tmp
mkfs.xfs /dev/VolGroup_tmp/LogVol00 
mount /dev/VolGroup_tmp/LogVol00 /mnt
xfsdump -J - / | xfsrestore -J - /dev/mnt
```
создаем раздел под /var
```sh
pvcreate /dev/sdd /dev/sde
vgcreate VolGroup01 /dev/sdd /dev/sde
lvcreate -l +100%FREE -m1 -n LogVol00 VolGroup01
mkfs.xfs /dev/VolGroup01/LogVol00
```
создаем раздел под /home 
```sh
pvcreate /dev/sdc
vgcreate VolGroup02 /dev/sdc
lvcreate -l +100%FREE -n LogVol00 VolGroup02
mkfs.xfs /dev/VolGroup02/LogVol00
```
уходим в режим восстановления
```sh
systemctl isolate rescue.target
```
##Не сработало. 
Вносим изменения в /etc/fstab
```sh
echo "/dev/mapper/VolGroup01-LogVol00 /var                    xfs     defaults        0 0" >> /etc/fstab
echo "/dev/mapper/VolGroup02-LogVol00 /home                   xfs     defaults        0 0" >> /etc/fstab
```
##Грузимся с iso. Загружаемся в RescueMode
Находим и подключаем все VG
```sh
vgscan
vgchange -ay 
```
Удаляем /
```sh
lvremove /dev/VolGroup00/LogVol00
```
Создаем новый раздел размеров 8G
```sh
lvcreate -L 8G -n LogVol00 VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol00 
```
Восстанавливаем / на новом разделе
```sh
mkdir /mnt/oldroot /mnt/newroot /mnt/newvar /mnt/newhome
mount /dev/VolGroup_tmp/LogVol00 /mnt/oldroot 
mount /dev/VolGroup00/LogVol00 /mnt/newroot
xfsdump -J - /mnt/oldroot | xfsrestore -J - /mnt/newroot
```
Переносим /var в новый раздел 
```sh
mount /dev/VolGroup01/LogVol00 /mnt/newvar
cp -aR /mnt/oldroot/var/* /mnt/newvar/
```
Переносим /home в новый раздел 
```sh
mount /dev/VolGroup02/LogVol00 /mnt/newhome
cp -aR /mnt/oldroot/home/* /mnt/newhome/
```
##перезагружаемся в боевую систему 
```sh
reboot
```
проверяем 
```sh 
NAME                           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                              8:0    0   40G  0 disk
├─sda1                           8:1    0    1M  0 part
├─sda2                           8:2    0    1G  0 part /boot
└─sda3                           8:3    0   39G  0 part
  ├─VolGroup00-LogVol00        253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01        253:1    0  1.5G  0 lvm  [SWAP]
sdb                              8:16   0   10G  0 disk
└─VolGroup_tmp-tmp             253:3    0   10G  0 lvm
sdc                              8:32   0    2G  0 disk
└─VolGroup02-LogVol00          253:2    0 1020M  0 lvm
sdd                              8:48   0    1G  0 disk
├─VolGroup01-LogVol00_rmeta_1  253:6    0    4M  0 lvm
│ └─VolGroup01-LogVol00        253:8    0 1016M  0 lvm
└─VolGroup01-LogVol00_rimage_1 253:7    0 1016M  0 lvm
  └─VolGroup01-LogVol00        253:8    0 1016M  0 lvm
sde                              8:64   0    1G  0 disk
├─VolGroup01-LogVol00_rmeta_0  253:4    0    4M  0 lvm
│ └─VolGroup01-LogVol00        253:8    0 1016M  0 lvm
└─VolGroup01-LogVol00_rimage_0 253:5    0 1016M  0 lvm
  └─VolGroup01-LogVol00        253:8    0 1016M  0 lvm
```  
/var на отдельном томе в mirror
/home на отдельном томе 
/ уменьшен до 8G

удаляем временный том 
```sh
lvremove /dev/VolGroup_tmp/LogVol00
vgremove VolGroup_tmp
pvremove /dev/sdb 
```

Генерируем файлы в /home 
```sh
 for i in 1 2 3 4 5; do truncate -s 2048 /home/file$i; done
```
проверяем
```sh 
ls /home
file1  file2  file3  file4  file5  vagrant
```
снимаем снэпшот
```sh
lvcreate -L 200G -s -n Home_Snap /dev/VolGroup02/LogVol00
```
удаляем файлы
```sh
for i in 1 3 5; do rm -f /home/file$i; done 
```
проверяем 
```sh
ls /home
file2  file4  vagrant
```
восстанавливаем из снэпшота
```sh
umount /home
```
ругается на занятость
```sh
umount: /home: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```
смотрим, что мешает
```sh
lsof /home
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash    1383 vagrant  cwd    DIR  253,2       74   68 /home/vagrant
sudo    1406    root  cwd    DIR  253,2       74   68 /home/vagrant
```
уходим в локальную консоль, и логинимся root
восстанавливаем из снэпшота
```sh
kill -9 1383 1406
umount /home
lvconvert --merge /dev/VolGroup02/Home_Snap
mount /home

```
проверяем
```sh
ls /home
file1  file2  file3  file4  file5  vagrant
``
