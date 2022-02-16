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

5. Включаем сжатие на каждой из файловых систем `otus1-4`
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
```

6. 