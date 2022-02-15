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

1. **_Запускаем машину и входим в неё по ssh - `vagrant up` и `vagrant ssh`_**.



























# **Введение**

Все ниже описанные действия производятся на компьютере с установленным `Windows 10`. Изначально производилась установка Centos 7 и Debian 11 в виртуальную машину через Virtual Box. Однако в процессе выполнения ДЗ выяснилось, что с Nested virtualization операции внутри виртуальной машины с установленным VirtualBox, Vagrant, Packer происходят в разы дольше, чем на хостовой машине. Соотвественно тестовый стенд был собран на хостовой машины с Windows 10.

Для выполнения работы потребуются следующие инструменты:

- **VirtualBox** - среда виртуализации, позволяет создавать и выполнять виртуальные машины;
- **Vagrant** - ПО для создания и конфигурирования виртуальной среды. В данном случае в качестве среды виртуализации используется *VirtualBox*;
- **Packer** - ПО для создания образов виртуальных машин;
- **GitHub Desktop** - система контроля версий

А так же аккаунты:

- **GitHub** - https://github.com/
- **Vagrant Cloud** - https://app.vagrantup.com


---
# **Установка ПО**
### **VirtualBox**
Скачиваем по следующей ссылке VirtualBox - https://download.virtualbox.org/virtualbox/6.1.32/VirtualBox-6.1.32-149290-Win.exe, а также Extension Pack - https://download.virtualbox.org/virtualbox/6.1.32/Oracle_VM_VirtualBox_Extension_Pack-6.1.32.vbox-extpack и запускаем их - Run as Administrator. Для того, чтобы в будущем управлять VirtualBox через PowerShell или Cmd необходимо выполнить одну из нижеописанных команд:

```
В Cmd
	SET PATH=%PATH%;C:\Program Files\Oracle\VirtualBox
Или в Powershell
	$env:PATH = $env:PATH + ";C:\Program Files\Oracle\VirtualBox"
```
Проверяем версию (в Powershell)

```
VBoxManage -version
```

### **Vagrant**
Скачиваем по следующей ссылке Vagrant - https://releases.hashicorp.com/vagrant/2.2.19/vagrant_2.2.19_x86_64.msi и запускаем их - Run as Administrator. Далее проверяем установку (в Powershell)

```
vagrant version
```

### **Packer**
Для установки Packer сначала устанавливаем CHOCOLATEY — менеджер пакетов в среде Windows по аналогии с apt-get в Linux Мире. Для его установки запускаем PowerShell - Run as Administrator и дальше выполняем следующую команду:

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```
После установки CHOCOLATEY устанавливаем с его помощью Packer и проверяем версию

```
choco install packer
packer version
```

### **GitHub Desktop**

Скачиваем по следующей ссылке GitHub Desktop -  https://central.github.com/deployments/desktop/desktop/latest/win32  и запускаем установщик- Run as Administrator.

---
# **Настройка ПО и выполнение практического задания**

### **Работа с GitHub Desktop**
После установки будем использовать графический интерфейс. Но сначала нам необходимо создать аккаунт на GitHub. После заведения аккаунта, входим под своим аккаунтом через браузер на https://github.com/ - авторизуемся. Дальше проходим по ссылке https://github.com/dmitry-lyutenko/manual_kernel_update и вверхнем правом углу нажимаем кнопку Fork. После этого уже в вашем аккаунте у вас появляется Pudlic repositorie - <user name / manual_kernel_update>. При необходимости переименовываем репозиторий - например "Otus-Pr-linux-HW1". Переходим в данный репозиторий, нажимаем кнопку Code - выбираем HTTPS - копируем предложенную ссылку.

На локальном компьютере запускаем GitHub Desktop - File - Clone Repository - URL - вставляем раннее скопированную ссылку - указываем локальную папку, где сохранить репозиторий - нажимаем Clone. Теперь у вас на компьютере локальная копия репозитория, с которым и продолжаем работать.

### **Создаём Box с обновлённым ядром (Centos 7.9.2009 / Kernel - 5.16.5)**
Для обучения, сначала проводим ручные операции по обновлению ядра, которые описаны в методичке - https://github.com/dmitry-lyutenko/manual_kernel_update/blob/master/manual/manual.md. Далее собираем собственный Box и выкладываем его на Vagrant Cloud. Все действия расписаны в вышеукзанной методичке, так что смысла в их copy-paste не вижу. Однако ниже приведу ошибки, которые появлялись, пока я пытался собрать сначала тестовую лабу с Nested virtualization на хостовой VM - Centos 7, а потом уже на хосте с Windows 10.

В ходе выполнения ДЗ появлялись следующие ошибки:


1. Проблема при подключении к машине по ssh - vagrant ssh, в связи с чем пришлось добавить в итоговый Vagrantfile (возможно было связанно и с другим):

```
# Problem wtih ssh
config.ssh.forward_agent = true
config.ssh.insert_key = false
```
2. При работe c Packer:

- `При переходе к запуску скриптов sh - пользователь vagrant не в sudoers`,
```
==> centos-7.7: Provisioning with shell script: scripts/stage-1-kernel-update.sh
   centos-7.7: >>> /etc/sudoers.d/vagrant: syntax error near line 1 <<<
   centos-7.7: sudo: parse error in /etc/sudoers.d/vagrant near line 1
   centos-7.7: sudo: no valid sudoers sources found, quitting
   centos-7.7: sudo: unable to initialize policy plugin
   centos-7.7: Provisioning step had errors: Running the cleanup provisioner, if present...
```

Для решения этой проблемы были изменены следующие строки в Vagrant.ks
```
- authconfig --enableshadow --passalgo=sha512
- user --name=vagrant --groups=vagrant --password=vagrant
- cat > /etc/sudoers.d/vagrant << EOF_sudoers_vagrant
- vagrant        ALL=(ALL)       NOPASSWD: ALL
- EOF_sudoers_vagrant

+ user --name=vagrant --groups=wheel --plaintext --password=vagrant
+ usermod -aG vagrant vagrant
+ echo "vagrant ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
```

- `Ошибка из-за временного интервала`,
```
==> centos-7.7: Timeout waiting for SSH.          
```

Для решения этой проблемы были изменены следующие строки в centos.json
```
#Изменение интервала ожидания подключения к Box
+- "ssh_wait_timeout": "30m",
#Увеличение ресурсов
+-      "vboxmanage": [
        [  "modifyvm",  "{{.Name}}",  "--memory",  "2048" ],
        [  "modifyvm",  "{{.Name}}",  "--cpus",  "4" ]
      ],
```

- `Ошибка из-за параметра "iso_checksum_type": "sha256",`,
```
==>Deprecated configuration key: 'iso_checksum_type'. Please call `packer fix`
against your template to update your template to be compatible with the current
version of Packer...
```

Решение - удаление строки в centos.json
```
-"iso_checksum_type": "sha256",
```

- `При работе с Nested Virtualization`,
```
==>VBoxManage error: VBoxManage: error: The virtual machine 'packer-centos-vm' has terminated unexpectedly during startup because of signal 6
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component MachineWrap, interface IMachine...          
```

Решение - добавление строки в centos.json
```
+ "headless": "true",
```

# **Заключение**

Итогом данного ДЗ стал собранный и загруженный в Vagrant Cloud образ  - ivashindd/centos-7-5, а также итоговый Vagrantfile, c помощью которого можно установить VM в VirualBox с
Centos 7-5.