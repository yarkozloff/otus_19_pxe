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
- Настроить доступ по HTTP для файлов из каталога /iso. Для этого подготовлен конфиг pxeboot.conf:
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
![image](https://user-images.githubusercontent.com/69105791/187760119-e56de623-c40c-43f5-904e-5168251c1a64.png)
Файлы доступны по сети

## Настройка TFTP-сервера
TFTP-сервер потребуется для отправки первичных файлов загрузки (vmlinuz, initrd.img и т. д.). Также реализуется через Ansible, для этого необходимо:
- Установить tftp-сервер
- Созать каталог, в котором будем хранить наше меню загрузки и создаем само меню:
```
default menu.c32
prompt 0
#Время счётчика с обратным отсчётом (установлено 15 секунд)
timeout 150
#Параметр использования локального времени
ONTIME local
#Имя «шапки» нашего меню
menu title OTUS PXE Boot Menu
#Описание первой строки
label 1
#Имя, отображаемое в первой строке
menu label ^ Graph install CentOS 8
#Адрес ядра, расположенного на TFTP-сервере
kernel /vmlinuz
#Адрес файла initrd, расположенного на TFTP-сервере
initrd /initrd.img
#Получаем адрес по DHCP и указываем адрес веб-сервера
append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8
label 2
menu label ^ Text install CentOS 8
kernel /vmlinuz
initrd /initrd.img
append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 text
label 3
menu label ^ rescue installed system
kernel /vmlinuz
initrd /initrd.img
append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 rescue
```
- Распаковать rpm пакет syslinux-tftpboot, достать от туда файлы и скопировать в директорию /var/lib/tftpboot/:
* pxelinux.0
* ldlinux.c32
* libmenu.c32
* libutil.c32
* menu.c32
* vesamenu.c32
- Также копировать файлы initrd.img и vmlinuz
- Перезапустить TFTP-сервер, добавив его в автозагрузку

## Настройка DHCP-сервера
Аналогично предыдущим модулям развертывание через ansible выглядит так:
- Установить DHCP-сервер
- Подготовить конфиг /etc/dhcp/dhcpd.conf:
```
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;
#Указываем сеть и маску подсети, в которой будет работать DHCP-сервер
subnet 10.0.0.0 netmask 255.255.255.0 {
#Указываем шлюз по умолчанию, если потребуется
#option routers 10.0.0.1;
#Указываем диапазон адресов
range 10.0.0.100 10.0.0.120;
class "pxeclients" {
match if substring (option vendor-class-identifier, 0, 9) =
"PXEClient";
#Указываем адрес TFTP-сервера
next-server 10.0.0.20;
#Указываем имя файла, который надо запустить с TFTP-сервера
filename "pxelinux.0";
}
```
- Перезапустить dhcpd и добавить в автозагрузку

## Проверка
Запускаем машину вручную из VBoxManage:

![image](https://user-images.githubusercontent.com/69105791/187791228-c13c7ee2-d041-4258-a166-2e6c0d6d3b15.png)

Нееет, за что. Службы запущены dhcp, tftpd, httpd работают, selinux выключен, страница с хоста доступна. Где-то на просторах интернета ннашлось почему оно возникает: 
```
https://docs.oracle.com/cd/E36784_01/html/E36800/more.html
...
DHCP-сервер предоставляет IP-адрес и местоположение программы начальной загрузки как часть ответа DHCP.
Если загрузочная программа не существует, загрузка клиента AI не может быть продолжена.  
```

