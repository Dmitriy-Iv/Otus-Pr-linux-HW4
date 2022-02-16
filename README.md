# **Описание**

В данном домашнем задании необходимо получить практические навыки по настройке тестовых пулов zfs, посмотреть основные команды работы с это файловой системой и её базовые возможности.

---

# **Подготовка VM** 

Для тестового стенда нам потребуется Vagrant файл из методички, часть которого относится к настройке и установки zfs в нашу VM.
```
	box.vm.provision "shell", inline: <<-SHELL
		#install zfs repo
		yum install -y http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
		#import gpg key
		rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
		#install DKMS style packages for correct work ZFS
		yum install -y epel-release kernel-devel zfs
		#change ZFS repo
		yum-config-manager --disable zfs
		yum-config-manager --enable zfs-kmod
		yum install -y zfs
		#Add kernel module zfs
		modprobe zfs
		#install soft
		yum install -y wget mc
	SHELL
```

---

# **Работа с ZFS**

1. Запускаем машину и входим в неё по ssh - `vagrant up` и `vagrant ssh`, далее переходим в окружение root - `sudo -i` и начинаем работать,
2. Проверяем список дисков
```
root@zfs ~]# lsblk
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
3. Создаём на 8 свободных дисках 4 пула с RAID1 (зеркало).
```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
```
4. Проверяем информацию о созданных пулах и дисках, которые входят в их состав.
```
[root@zfs ~]# zpool list && zpool status
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  94.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
  pool: otus1
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors
```
5. Включаем сжатие на каждой из файловых систем `otus1-4`.
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
```
6. Проверям, что все системы имеют разные методы сжатия
```
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
7. Тестируем сжатие, качаем один и тот же текстовый файл и раскладываем его на разные пулы.
```
[root@zfs ~]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2022-02-16 00:37:44--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40784369 (39M) [text/plain]
Saving to: ‘/otus1/pg2600.converter.log’

100%[=====================================================================================================================================================================================================================================>] 40,784,369  3.76MB/s   in 18s

2022-02-16 00:38:03 (2.16 MB/s) - ‘/otus1/pg2600.converter.log’ saved [40784369/40784369]

--2022-02-16 00:38:03--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40784369 (39M) [text/plain]
Saving to: ‘/otus2/pg2600.converter.log’

100%[=====================================================================================================================================================================================================================================>] 40,784,369  2.93MB/s   in 17s

2022-02-16 00:38:21 (2.24 MB/s) - ‘/otus2/pg2600.converter.log’ saved [40784369/40784369]

--2022-02-16 00:38:21--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40784369 (39M) [text/plain]
Saving to: ‘/otus3/pg2600.converter.log’

100%[=====================================================================================================================================================================================================================================>] 40,784,369  1.81MB/s   in 24s

2022-02-16 00:38:46 (1.59 MB/s) - ‘/otus3/pg2600.converter.log’ saved [40784369/40784369]

--2022-02-16 00:38:46--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40784369 (39M) [text/plain]
Saving to: ‘/otus4/pg2600.converter.log’

100%[=====================================================================================================================================================================================================================================>] 40,784,369  2.27MB/s   in 20s

2022-02-16 00:39:07 (1.94 MB/s) - ‘/otus4/pg2600.converter.log’ saved [40784369/40784369]
```
8. Проверяем самый оптимальный метод сжатия - то есть наименьшее количество блоков занятых этим файлов на разных пулах - параметр `total`. Определяем, что это на пуле otus3.
```
[root@zfs ~]# ls -l /otus*
/otus1:
total 22015
-rw-r--r--. 1 root root 40784369 Feb  2 09:01 pg2600.converter.log

/otus2:
total 17970
-rw-r--r--. 1 root root 40784369 Feb  2 09:01 pg2600.converter.log

/otus3:
total 10948
-rw-r--r--. 1 root root 40784369 Feb  2 09:01 pg2600.converter.log

/otus4:
total 39856
-rw-r--r--. 1 root root 40784369 Feb  2 09:01 pg2600.converter.log
```
9. Проверяем занимаемое место этим файлом на разных пулах и коэффициент сжатия, для того чтобы подтвердить - что gzip-9 на otus3 самый эффективный.
с
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.64x                  -
otus4  compressratio         1.00x                  -
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.5M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.0M   313M     38.9M  /otus4
```
10. Теперь посмотрим на настройки zpool. Для этого сначала скачиваем файл архив пула.
```
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
--2022-02-16 18:00:06--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Resolving drive.google.com (drive.google.com)... 173.194.221.194, 2a00:1450:4010:c0a::c2
Connecting to drive.google.com (drive.google.com)|173.194.221.194|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]
--2022-02-16 18:00:06--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/83e2aju958nbbcf1mcukh8gses10hh88/1645034400000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download [following]
Warning: wildcards not supported in HTTP.
--2022-02-16 18:00:10--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/83e2aju958nbbcf1mcukh8gses10hh88/1645034400000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download
Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 142.251.1.132, 2a00:1450:4010:c1e::84
Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|142.251.1.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/x-gzip]
Saving to: ‘archive.tar.gz’

100%[=====================================================================================================================================================================================================================================>] 7,275,140   11.4MB/s   in 0.6s

2022-02-16 18:00:11 (11.4 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]
```
11. Разархивируем его.
```
[root@zfs ~]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
12. Смотрим, можно ли сделать импорт пула
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
```
13. Проверяем ещё раз, что такого пула пока ещё нет у нас в системе 
```
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  51.5M   428M        -         -     2%    10%  1.00x    ONLINE  -
otus2   480M  51.5M   428M        -         -     1%    10%  1.00x    ONLINE  -
otus3   480M  51.5M   429M        -         -     1%    10%  1.00x    ONLINE  -
otus4   480M  51.6M   428M        -         -     1%    10%  1.00x    ONLINE  -
```
14. Импортируем пул и проверяем снова
```
[root@zfs ~]# zpool list && zpool status
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus    480M  2.09M   478M        -         -     0%     0%  1.00x    ONLINE  -
otus1   480M  51.5M   428M        -         -     2%    10%  1.00x    ONLINE  -
otus2   480M  51.5M   428M        -         -     1%    10%  1.00x    ONLINE  -
otus3   480M  51.5M   429M        -         -     1%    10%  1.00x    ONLINE  -
otus4   480M  51.6M   428M        -         -     1%    10%  1.00x    ONLINE  -
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

  pool: otus1
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors
```
15. Как мы видим, у нас появился новый пул otus. Смотрим вывод всех его параметров сначала через zpool, а потом через zfs
```
[root@zfs ~]# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupditto                     0                              default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      9392072699242269180            -
otus  autotrim                       off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local

[root@zfs ~]# zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```
16. Запрашивая через - `zfs get <PARAMETRS> otus`


```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
[root@zfs ~]# zfs get version otus
NAME  PROPERTY  VALUE    SOURCE
otus  version   5        -
```
