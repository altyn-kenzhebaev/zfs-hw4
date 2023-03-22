# ZFS
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/zfs-hw4.git`
В текущей директории появится папка с именем репозитория. В данном случае hw-1. Ознакомимся с содержимым:
```
cd zfs-hw4
ls -l
README.md
Vagrantfile
```
Здесь:
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ, выводим список дисков:
```
vagrant up
[vagrant@zfs ~]$ sudo -i
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 
```
### Определить алгоритм с наилучшим сжатием
Создадим 4 пула типа зеркало по 2 диска в каждом:
```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
```
На каждом применим свой алгоритм сжатия:
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
```
Проверим с помощью команды `zfs get all | grep compression`:
```
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
Скачаем файл и разместим копию на каждом томе:
```
[root@zfs ~]# wget -O War_and_Peace.txt http://www.gutenberg.org/ebooks/2600.txt.utf-8
--2023-03-22 08:34:54--  http://www.gutenberg.org/ebooks/2600.txt.utf-8
Resolving www.gutenberg.org (www.gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:80... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://www.gutenberg.org/ebooks/2600.txt.utf-8 [following]
--2023-03-22 08:34:54--  https://www.gutenberg.org/ebooks/2600.txt.utf-8
Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://www.gutenberg.org/cache/epub/2600/pg2600.txt [following]
--2023-03-22 08:34:55--  https://www.gutenberg.org/cache/epub/2600/pg2600.txt
Reusing existing connection to www.gutenberg.org:443.
HTTP request sent, awaiting response... 200 OK
Length: 3359372 (3.2M) [text/plain]
Saving to: ‘War_and_Peace.txt’

100%[=========================================================================================================================================================================>] 3,359,372    457KB/s   in 9.2s   

2023-03-22 08:35:04 (357 KB/s) - ‘War_and_Peace.txt’ saved [3359372/3359372]

[root@zfs ~]# for i in {1..4}; do cp War_and_Peace.txt /otus$i/; done
```
Проверим что файл скопирован, и тут же можно понять какой алгоритм сжатия наиболее эффективен (otus3 с алгоритмом gzip-9):
```
[root@zfs ~]# ls -lh /otus*

/otus1:
total 24M
-rw-r--r--. 1 root root  40M Mar  2 09:17 pg2600.converter.log
-rw-r--r--. 1 root root 3.3M Mar 22 08:35 War_and_Peace.txt

/otus2:
total 20M
-rw-r--r--. 1 root root  40M Mar  2 09:17 pg2600.converter.log
-rw-r--r--. 1 root root 3.3M Mar 22 08:35 War_and_Peace.txt

/otus3:
total 12M
-rw-r--r--. 1 root root  40M Mar  2 09:17 pg2600.converter.log
-rw-r--r--. 1 root root 3.3M Mar 22 08:35 War_and_Peace.txt

/otus4:
total 43M
-rw-r--r--. 1 root root  40M Mar  2 09:17 pg2600.converter.log
-rw-r--r--. 1 root root 3.3M Mar 22 08:35 War_and_Peace.txt
```
Также наилучший степень сжатия можем проверить следующей командой:
```
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.65x                  -
otus4  compressratio         1.00x                  -
```
### Определение настроек пула
Для определения настроек пула воспользуемся скачанным архивом по методичке, скачаем и разархивируем:
```
wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
tar -xzvf archive.tar.gz
```
Проверим данный пул, если все хорошо импортируем его:
```
[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status otus
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
```
Командой `zfs get all otus` можем вывести все настройки пула, или вывести каждую настройку отдельно:
```
[root@zfs ~]# zfs get available otus    # Оставшееся  свободное место 
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
[root@zfs ~]# zfs get readonly otus     # Является ли ФС только на чтение
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
[root@zfs ~]# zfs get recordsize  otus  # Размер recordsize
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
[root@zfs ~]# zfs get compression otus  # Алгоритм компресии
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
[root@zfs ~]# zfs get checksum otus     # Тип используемой контрольной суммы 
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
### Поиск сообщения от преподавателя (навыки восстановления snapshot и переноса файла)
Скачаем файл снапшота и восстановим его на диск Otus:
```
wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download"
zfs receive otus/test@today < otus_task2.file
```
Ищем файл и выводим secret_message:
```
[root@zfs ~]# find /otus/test -name "secret_message" | xargs cat
https://github.com/sindresorhus/awesome
```