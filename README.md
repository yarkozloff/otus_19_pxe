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
