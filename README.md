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
**__[root@zfs ~]# lsblk__**
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