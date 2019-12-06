# **Настройка кластера для поддержки функции Routing Behind MS vUGW9811/UGW9811/GGSN9811 компании Huawei**

Настройка выполняется с использованием сценария ansible, по результатам работы которого получаем работающий RADIUS сервер с возможностью работы в режиме active/standby, а также режиме active/active в виду использования сервера LDAP с репликацией данных между нодами.

## **Начальные требования**

Для кластера необходимо два сервера с минимальной инсталляцией CentOS7 с настроенной сетевой подсистемой для обеспечения связности с интерфейсом Gif GGSN/PGW, сетью O&M и прямым интерфейсом между серверами кластера. Для работы с ansible необходим доступ root на сервер через ssh с авторизацией по ключу, но всё на усмотрение администратора системы.

## **Порядок установки**

### 1. Изменение параметров ansible сценария

В файле __inventory__ указываем адреса O&M master и slave сервера.

```
[master]
<master.ip>

[slave]
<slave.ip>

[nodes:children]
master
slave
```
В файле __group_vars/nodes__ указываем создаваемый LDAP домен, пароль LDAP администратора в открытом и закрытом виде, адрес RADIUS  сервера для режима active/standby, VRRP ID и пароль для авторизации VRRP пира, а так же список RADIUS клиентов. Адресация cluster-internal - это сеть на интерфейсе между серверами кластера, cluster-external - это адресация внешнего интерфейса RADIUS сервера.

```
---
ldap_domain: 'dc=telecom,dc=ru'
ldap_root_pw_plain: X44Yc4NJ
ldap_root_pw_encrypted: '{SSHA}eKlFDrGH9AyVHgMxFZDQzymj94p5li8D'

vrrp_ext_vip: <external.vif.ip>/32
vrrp_ext_vrid: 107
vrrp_pass: thKe60V4

radius_secret: telecomdv
radius_client:
 - { name: cluster-internal, ip: 100.0.0.0/30, secret: telecomdv }
 - { name: cluster-external, ip: 200.0.0.0/28, secret: telecomdv }
 - { name: vugw01, ip: 10.27.100.200/32, secret: huawei }
 - { name: vugw02, ip: 10.38.100.200/32, secret: huawei }
```
В файле __host_vars/<master.ip>__ указываем имя Master ноды, название внешнего интерфейса RADIUS сервера.

```
---
node_name: <master.name>
ldap_host_id: 0
ldap_provider_host: <slave.internal.ip>
vrrp_ext_if: <external.if.name>
vrrp_priority: 200
vrrp_state: MASTER
```
В файле __host_vars/<slave.ip>__ указываем имя Slave ноды, название внешнего интерфейса RADIUS сервера.

```
---
node_name: <slave.name>
ldap_host_id: 1
ldap_provider_host: <master.internal.ip>
vrrp_ext_if: <external.if.name>
vrrp_priority: 100
vrrp_state: BACKUP
```

### 2. Запуск ansible сценария

```
ansible-playbook -i inventory rbms.yml
```

### 3. Проверка работы сервера после настройки

Для проверки работоспособности сервера в директории __/root__ создаётся скрипт `peertest.sh` проверяющий работоспособоность соседнего сервера кластера.

Пример вывода с Master ноды после инсталляции:

```
[root@radius-rbms-1 ~]# ./peertest.sh 

### Get routes by IP ###

Sent Access-Request Id 177 from 0.0.0.0:35095 to 100.0.0.2:1812 length 97
        User-Name = "79240123456"
        User-Password = "password"
        Framed-IP-Address = 10.27.216.90
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79240123456"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 177 from 100.0.0.2:1812 to 0.0.0.0:0 length 46
        Framed-Route = "10.0.0.0/24"
        Framed-Route = "10.0.1.0/24"

### Get routes by MSISDN ###

Sent Access-Request Id 56 from 0.0.0.0:56074 to 100.0.0.2:1812 length 91
        User-Name = "79246543210"
        User-Password = "password"
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79246543210"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 56 from 100.0.0.2:1812 to 0.0.0.0:0 length 33
        Framed-Route = "10.0.3.0/24"

### Get routes by unknown MSISDN ###

Sent Access-Request Id 173 from 0.0.0.0:59217 to 100.0.0.2:1812 length 91
        User-Name = "79240123456"
        User-Password = "password"
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79240123456"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 173 from 100.0.0.2:1812 to 0.0.0.0:0 length 20
[root@radius-rbms-1 ~]# 
```

Пример вывода с Slave ноды после инсталляции:

```
[root@radius-rbms-2 ~]# ./peertest.sh 

### Get routes by IP ###

Sent Access-Request Id 254 from 0.0.0.0:38758 to 100.0.0.1:1812 length 97
        User-Name = "79240123456"
        User-Password = "password"
        Framed-IP-Address = 10.27.216.90
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79240123456"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 254 from 100.0.0.1:1812 to 0.0.0.0:0 length 46
        Framed-Route = "10.0.0.0/24"
        Framed-Route = "10.0.1.0/24"

### Get routes by MSISDN ###

Sent Access-Request Id 245 from 0.0.0.0:57552 to 100.0.0.1:1812 length 91
        User-Name = "79246543210"
        User-Password = "password"
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79246543210"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 245 from 100.0.0.1:1812 to 0.0.0.0:0 length 33
        Framed-Route = "10.0.3.0/24"

### Get routes by unknown MSISDN ###

Sent Access-Request Id 250 from 0.0.0.0:45957 to 100.0.0.1:1812 length 91
        User-Name = "79240123456"
        User-Password = "password"
        NAS-Port-Type = Virtual
        Calling-Station-Id = "79240123456"
        Called-Station-Id = "test.dv"
        Service-Type = Framed-User
        Framed-Protocol = GPRS-PDP-Context
        Cleartext-Password = "password"
Received Access-Accept Id 250 from 100.0.0.1:1812 to 0.0.0.0:0 length 20
[root@radius-rbms-2 ~]# 
```

## **Управление абонентскими данными**

Для управления данными в БД LDAP используется скрипт `rbmscli`. При запуске без параметров или с ошибочным набором параметров выдаётся инструкция по использованию.

```
[root@radius-rbms-1 ~]# rbmscli 

Usage:

    rbmscli --show --apn all
    rbmscli --show --apn <apn.name>

    ### Add APN.

    rbmscli --add --apn <apn.name>

    ### Add MSISDN with APN and route assigned to MSISDN.

    rbmscli --add --apn <apn.name> --msisdn <msisdn>
    rbmscli --add --apn <apn.name> --msisdn <msisdn> --route <route>

    ### Add MS ip-address with APN and route assigned to this ip-address.

    rbmscli --add --apn <apn.name> --nexthop <ms ip-address>
    rbmscli --add --apn <apn.name> --nexthop <ms ip-address> --route <route>

    ### Delete. All delete operations are recursive.

    rbmscli --delete --apn <apn.name>
    rbmscli --delete --apn <apn.name> --msisdn <msisdn>
    rbmscli --delete --apn <apn.name> --msisdn <msisdn> --route <route>
    rbmscli --delete --apn <apn.name> --nexthop <ms ip-address>
    rbmscli --delete --apn <apn.name> --nexthop <ms ip-address> --route <route>

[root@radius-rbms-1 ~]# 
```

Логика работы позволяет добавлять маршрут как по номеру абонента так и по его IP адресу. Для добавления маршрута первоначально должне быть создан APN и добавлен в него номер либо IP адрес абонента. Возможно добавление обоих параметров, но при работе приоритетными будут маршруты прописанные по IP адресу.

### Примеры работы с `rbmscli` ###

#### Посмотреть список заведённых APN

```
[root@radius-rbms-1 ~]# rbmscli --show --apn all
apn test.dv
[root@radius-rbms-1 ~]# 
```

#### Просмотр конфигурации конкретного APN

```
[root@radius-rbms-1 ~]# rbmscli --show --apn test.dv
apn test.dv
!
 10.27.216.90
  10.0.0.0/24
  10.0.1.0/24
!
 10.27.216.91
  10.0.2.0/24
!
 79246543210
  10.0.3.0/24

[root@radius-rbms-1 ~]# 
```

#### Добавить APN,абонентский номер и маршрут на этот номер

```
[root@radius-rbms-1 ~]# rbmscli --add --apn test.volga
adding new entry "ou=test.volga,ou=apn,dc=telecom,dc=ru"

adding new entry "ou=msisdn,ou=test.volga,ou=apn,dc=telecom,dc=ru"

adding new entry "ou=route,ou=test.volga,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --add --apn test.volga --msisdn 79270123456
adding new entry "cn=79270123456,ou=msisdn,ou=test.volga,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --add --apn test.volga --msisdn 79270123456 --route 10.0.0.0/24
modifying entry "cn=79270123456,ou=msisdn,ou=test.volga,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --show --apn test.volga
apn test.volga
!
 79270123456
  10.0.0.0/24

[root@radius-rbms-1 ~]#
```

#### Добавить APN,абонентский IP адрес и маршрут на этот адрес

```
[root@radius-rbms-1 ~]# rbmscli --add --apn test.nsk
adding new entry "ou=test.nsk,ou=apn,dc=telecom,dc=ru"

adding new entry "ou=msisdn,ou=test.nsk,ou=apn,dc=telecom,dc=ru"

adding new entry "ou=route,ou=test.nsk,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --add --apn test.nsk --nexthop 172.16.0.1
adding new entry "cn=172.16.0.1,ou=route,ou=test.nsk,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --add --apn test.nsk --nexthop 172.16.0.1 --route 20.0.0.0/24
modifying entry "cn=172.16.0.1,ou=route,ou=test.nsk,ou=apn,dc=telecom,dc=ru"

[root@radius-rbms-1 ~]# rbmscli --show --apn test.nsk
apn test.nsk
!
 172.16.0.1
  20.0.0.0/24

[root@radius-rbms-1 ~]# 
```

#### Удалить номер с указанного APN

```
[root@radius-rbms-1 ~]# rbmscli --delete --apn=test.volga --msisdn 79270123456    
[root@radius-rbms-1 ~]# rbmscli --show --apn=test.volga                       
apn test.volga

[root@radius-rbms-1 ~]#
```

#### Удалить IP адрес с указанного APN

```
[root@radius-rbms-1 ~]# rbmscli --delete --apn=test.nsk --nexthop 172.16.0.1    
[root@radius-rbms-1 ~]# rbmscli --show --apn=test.nsk                       
apn test.nsk

[root@radius-rbms-1 ~]# 
```

#### Удаление APN

Операция удаления APN рекурсивная, т.е. если удалить APN, то будут удалены все связанные с ним данные.

```
[root@radius-rbms-1 ~]# rbmscli --delete --apn test.nsk
[root@radius-rbms-1 ~]# rbmscli --show --apn all
apn test.dv
apn test.volga
[root@radius-rbms-1 ~]# rbmscli --delete --apn test.volga
[root@radius-rbms-1 ~]# rbmscli --show --apn all         
apn test.dv
[root@radius-rbms-1 ~]#
```