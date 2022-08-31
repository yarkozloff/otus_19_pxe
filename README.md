# Vagrant-стенд c PXE
## Описание домашнего задания
- Следуя шагам из документа https://docs.centos.org/en-US/8-docs/advanced-install/assembly_preparing-for-a-network-install установить и настроить загрузку по сети для дистрибутива CentOS8. В качестве шаблона воспользуйтесь репозиторием https://github.com/nixuser/virtlab/tree/main/centos_pxe.
- Поменять установку из репозитория NFS на установку из репозитория HTTP.
- Настроить автоматическую установку для созданного kickstart файла (*) Файл загружается по HTTP.
- *автоматизировать процесс установки Cobbler cледуя шагам из документа https://cobbler.github.io/quickstart/.
Формат сдачи ДЗ - vagrant + ansible
## Среда выполнения:
```
root@yarkozloff:/otus/pxe# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
root@yarkozloff:/otus/pxe# vboxmanage --version
6.1.26_Ubuntur145957
root@yarkozloff:/otus/pxe#  vagrant --version
Vagrant 2.3.0
root@yarkozloff:/otus/pxe# ansible --version
ansible [core 2.12.4]
```
## Теория
PXE (Preboot eXecution Environment) — это набор протоколов, которые позволяют загрузить хост из сети. Для загрузки будет использоваться сетевая карта хоста.

Для PXE требуется:
- Со стороны клиента (хоста на котором будем устанавливать или загружать ОС):
  * Cетевая карта, которая поддерживает стандарт PXE
- Со стороны сервера:
  * DHCP-сервер
  * TFTP-сервер
  
TFTP (Trivial File Transfer Protocol) — простой протокол передачи файлов, используется главным образом для первоначальной загрузки бездисковых рабочих станций. Основная задача протокола TFTP — отправка указанных файлов клиенту.

DHCP (Dynamic Host Configuration Protocol) — протокол динамической настройки узла, позволяет сетевым устройствам автоматически получать IP-адрес и другие параметры, необходимые для работы в сети TCP/IP. (67 порт на сервере и 68 порт на клиенте)

## Подготовка Vagrantfile
Будем использовать локально загруженный вагрант бокс bento/centos-8.4. На хостовую машину потребовалось выделить побольше ресурсов, мне хватило 5.7гб ОЗУ и 5 ядер. Далее необходимо было прокинуть порт, в машине pxeserver это 80, на хостовой 8081. После поднятия с конфигурацией из вложения, получим ошибку, так как на Pxeclient настроена загрузка по сети:
```
==> pxeclient: Preparing network interfaces based on configuration...
    pxeclient: Adapter 1: nat
    pxeclient: Adapter 2: hostonly
==> pxeclient: Forwarding ports...
    pxeclient: 22 (guest) => 2200 (host) (adapter 1)
There was an error while executing `VBoxManage`, a CLI used by Vagrant
for controlling VirtualBox. The command and stderr is shown below.

Command: ["modifyvm", "62551d5d-e65e-49d0-8d52-099f1731f5f4", "--natpf1", "ssh,tcp,127.0.0.1,2200,,22"]

Stderr: VBoxManage: error: A NAT rule of this name already exists
VBoxManage: error: Details: code NS_ERROR_INVALID_ARG (0x80070057), component NATEngineWrap, interface INATEngine, callee nsISupports
VBoxManage: error: Context: "AddRedirect(Bstr(strName).raw(), proto, Bstr(strHostIp).raw(), RTStrToUInt16(strHostPort), Bstr(strGuestIp).raw(), RTStrToUInt16(strGuestPort))" at line 1923 of file VBoxManageModifyVM.cpp
```
## Настройка Web-сервера
Для того, чтобы отдавать файлы по HTTP нам потребуется настроенный веб-сервер. Делаем сразу через ansible. 
- Так как у CentOS 8 закончилась поддержка, для установки пакетов нам потребуется поменять репозиторий, изменив его в конфигах /etc/yum.repos.d
- Установить Web-сервер Apache
- Скачать образ CentOS 8.4.2150 (таумаут для ансибл пришлось увеличить до 6000)
- Смонтировать скачанный образ в /mnt
- Создать каталог /iso и скопировать содержимое /mnt (ставим права 0755)
- Настроить доступ по HTTP для файлов из каталога /iso. Для этого подготовлен конфиг:
```
IndexOptions FancyIndexing HTMLTable VersionSort

Alias /centos8 /iso
<Directory "/iso">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```
- Перезапустить сервис httpd, добавив его в автозагрузку

Проверка с хост машины:
