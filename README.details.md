## Администрирование трехзвенной архитектуры на примере Django-приложений

В область администрирования входят:
* база данных (представлен PostgreSQL)
* сервер приложений (представлен комплексом из Nginx, Gunicorn, Django)

Также включает
* сервер журналирования
* сервер мониторинга
* сервер бекапирования

Все базируется на centos/7.

```text
LEGENDS:
# - network port-many commutators
< ** > - routers with eth-cards hardware
[ ** ] - servers hardware
( ** ) - packagr infrastructure

                     .-~~~-.
             .- ~ ~-(       )_ _~ -._
            \       INTERNET        .' 
              ~- . ___________ . -~'
                        |
                        |
               < Internet Router > :8080,:8443
                        |
                        #
:80 -> :443             |
[ WebServer ]--#--< Central Router >
   (Nginx)         |      |      |
                   #      |      #
                   |      #      |       
        < AppRouter >     |    [ ManageServer ]
          |               |     (rsyslog aggr) 
          |               |     
          |          < dbRouter >
          |           |        | 
[ AppServer ]         #        #     
  (Gunicorn)          |        |   
   (Django)     [ DbServer ]   |
                    (PG)       |
                               |
                       [ BackupDbServer ]
                             (***)
```

### Vagrant

Виртуалки базируются на centos/7.

```shell
cd ./vm/
vagrant destroy -f
vagrant up
```

Использую библиотеку для автоматического формирования Ansible-файла inventory


[details --no-link]:[импорт Vagrant хостов в Ansible inventory](./vm/v2a.py)

```shell
python3 v2a.py -o ../ansible/inventories/hosts
```

Для его использования можно сделать

```shell
pip3 install -r requirements.txt
```

Применил хранение настроек в переменной, для однотипной задачи прописывания шлюзов на интерфейсах

[details --no-link]:[Параметры шлюзов](./ansible/roles/routing/vars/main.yml)

Задача, использующая специальную ansible-переменную `inventory_hostname`, "знающую" для какого инвентори-хоста работает в данный момент плейбук.

[details --no-link]:[Задача](./ansible/roles/routing/tasks/main.yml)

Аналогично в рамках демонстрации работоспособности применил `loop` и `inventory_hostname` 

[details --no-link]:[Задача](./ansible/roles/intermediateTestOfNetworkConnectivity/tasks/main.yml)

Я отказался от NetworkManager в пользу классики, так как с ним у меня возникли непонятки по маршрутам в NAT

```shell
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl enable network.service
systemctl start network.service
```

### Ansible

```shell
cd ./ansible
```

#### Предварительно

На стенде трафик маршрутизируется исключительно в IPv4. Если не сделать нижеследующее, то у YUM будет отсутствовать доступ к репозиториям, так как по дефолту он в IPv6.

```shell
ansible-playbook playbooks/disable_ipv6.yml  > ../reports/playbooks/disable_ipv6.txt
```

[details --no-link]:[playbooks/disable_ipv6.yml](./reports/playbooks/disable_ipv6.txt)

#### Общее. Настройка сети

Одной командой настраиваем сеть инфраструктуры 

```shell
ansible-playbook playbooks/chrony.yml --tags deploy > ../reports/playbooks/chrony.txt
```

[details --no-link]:[playbooks/chrony.yml](./reports/playbooks/chrony.txt)

```shell
ansible-playbook playbooks/disable_ipv6.yml --tags deploy  > ../reports/playbooks/disable_ipv6.txt
```

[details --no-link]:[playbooks/disable_ipv6.yml](./reports/playbooks/disable_ipv6.txt)

```shell
ansible-playbook playbooks/routing.yml --tags deploy  > ../reports/playbooks/routing.txt
```

[details --no-link]:[playbooks/routing.yml](./reports/playbooks/routing.txt)

Доступ вовне идет через один узел, которому нужно об этом "напомнить"

```shell
ansible-playbook playbooks/exit_node.yml --tags deploy > ../reports/playbooks/exit_node.txt
```

[details --no-link]:[playbooks/exit_node.yml](./reports/playbooks/exit_node.txt)

```shell
ansible-playbook playbooks/intermediateTestOfNetworkConnectivity.yml  > ../reports/playbooks/intermediateTestOfNetworkConnectivity.txt
```

[details --no-link]:[playbooks/intermediateTestOfNetworkConnectivity.yml](./reports/playbooks/intermediateTestOfNetworkConnectivity.txt)


[details --no-link]:[iptables --table filter --list](./reports/tests/exit_node-iptables --table filter --list.json)
[details --no-link]:[iptables --table nat --list](./reports/tests/exit_node-iptables --table nat --list.json)

#### Общее. Проверка работоспособности сети

Произвожу проверку внутрисетевой связности от "сервера" к "серверу", используя пакет `traceroute`:

```shell
traceroute 8.8.8.8       # GOOGLE
traceroute 192.168.255.1 # internetRouter
traceroute 192.168.255.2 # centralRouter
traceroute 192.168.0.2   # webServer
traceroute 192.168.2.194 # appServer
traceroute 192.168.1.194 # dbServer
traceroute 192.168.0.66  # monitoringServer
```


Но обернуто в Anible

```shell
ansible-playbook playbooks/intermediateTestOfNetworkConnectivity.yml > ../reports/playbooks/intermediateTestOfNetworkConnectivity.txt
```

[details --no-link]:[intermediateTestOfNetworkConnectivity.yml](./reports/playbooks/intermediateTestOfNetworkConnectivity.txt)

##### internetRouter

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/internetRouter - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/internetRouter - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/internetRouter - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/internetRouter - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/internetRouter - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/internetRouter - traceroute 192.168.1.194.txt)

#### centralRouter

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/centralRouter - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/centralRouter - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/centralRouter - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/centralRouter - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/centralRouter - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/centralRouter - traceroute 192.168.1.194.txt)

#### webServer

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/webServer - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/webServer - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/webServer - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/webServer - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/webServer - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/webServer - traceroute 192.168.1.194.txt)

#### appRouter

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/appRouter - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/appRouter - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/appRouter - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/appRouter - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/appRouter - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/appRouter - traceroute 192.168.1.194.txt)

#### appServer

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/appServer - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/appServer - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/appServer - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/appServer - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/appServer - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/appServer - traceroute 192.168.1.194.txt)

#### dbRouter

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/dbRouter - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/dbRouter - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/dbRouter - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/dbRouter - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/dbRouter - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/dbRouter - traceroute 192.168.1.194.txt)

#### dbBackupServer

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/dbBackupServer - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/dbBackupServer - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/dbBackupServer - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/dbBackupServer - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/dbBackupServer - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/dbBackupServer - traceroute 192.168.1.194.txt)

#### monitoringServer

[details --no-link]:[traceroute 8.8.8.8](./reports/tests/routing/monitoringServer - traceroute 8.8.8.8.txt)

[details --no-link]:[traceroute 192.168.255.1](./reports/tests/routing/monitoringServer - traceroute 192.168.255.1.txt)

[details --no-link]:[traceroute 192.168.255.2](./reports/tests/routing/monitoringServer - traceroute 192.168.255.2.txt)

[details --no-link]:[traceroute 192.168.0.2](./reports/tests/routing/monitoringServer - traceroute 192.168.0.2.txt)

[details --no-link]:[traceroute 192.168.2.194](./reports/tests/routing/monitoringServer - traceroute 192.168.2.194.txt)

[details --no-link]:[traceroute 192.168.1.194](./reports/tests/routing/monitoringServer - traceroute 192.168.1.194.txt)

#### Настройка бд

```shell
ansible-playbook playbooks/postgresql.yml --tags deploy > ../reports/playbooks/postgresql.txt
```

[details --no-link]:[playbooks/postgresql.yml](./reports/playbooks/postgresql.txt)

```shell
ansible-playbook playbooks/intermediateTestOfDbAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfDbAvailabilityFromExternalHost.txt
```

[details --no-link]:[playbooks/intermediateTestOfDbAvailabilityFromExternalHost.yml](./reports/playbooks/intermediateTestOfDbAvailabilityFromExternalHost.txt)


[details --no-link]:[local access check](./reports/tests/postresql-dbServer-local_access_check-psql--c-SELECT-version()--U-admin--h-127.0.0.1-project--W.txt)
[details --no-link]:[remote access test from dbRouter](./reports/tests/postgresql-dbRouter-remote-access-test.txt)
[details --no-link]:[remote access test from appServer](./reports/tests/postgresql-appServer-remote-access-test.txt)


#### Настройка nginx

```shell
ansible-playbook playbooks/nginx.yml --tags deploy_django > ../reports/playbooks/nginx.txt
ansible-playbook playbooks/intermediateTestOfWebAvailabilityFromExternalHost.yml> ./reports/playbooks/intermediateTestOfWebAvailabilityFromExternalHost.txt
```

[details --no-link]:[playbooks/nginx.yml](./reports/playbooks/nginx.txt)

[details --no-link]:[playbooks/intermediateTestOfWebAvailabilityFromExternalHost.yml](./reports/playbooks/intermediateTestOfWebAvailabilityFromExternalHost.txt)

#### Настройка Django

```shell
ansible-playbook playbooks/django.yml --tags deploy > ../reports/playbooks/django.txt
ansible-playbook playbooks/intermediateTestOfAppAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfAppAvailabilityFromExternalHost.txt
```

[details --no-link]:[playbooks/django.yml](./reports/playbooks/django.txt)
[details --no-link]:[playbooks/intermediateTestOfAppAvailabilityFromExternalHost.yml](./reports/playbooks/intermediateTestOfAppAvailabilityFromExternalHost.txt)

#### Настройка логгирования

```shell
ansible-playbook playbooks/rsyslog_server.yml --tags deploy > ../reports/playbooks/rsyslog_server.txt
ansible-playbook playbooks/rsyslog_clients.yml --tags clients > ../reports/playbooks/rsyslog_clients.txt
```

[details --no-link]:[playbooks/rsyslog_server.yml](./reports/playbooks/rsyslog_server.txt)
[details --no-link]:[playbooks/rsyslog_server.yml](./reports/playbooks/rsyslog_server.txt)

### Для справки

```shell
python3 details.py template.README.md 
```
### Не забыть рассказать

* про словари и inventory name
* сохранение результатов
* шаблон отчета (шаблонизирование)
* про expect
* показать что не отрабатывают CSS
* экстра var
* разумная достаточность на примере IPtables без инкапсуляции

### Запустить и забыть

```shell
cd ../vm/
vagrant destroy -f
vagrant up
python3 v2a.py -o ../ansible/inventories/hosts
cd ../ansible

ansible-playbook playbooks/chrony.yml --tags deploy > ../reports/playbooks/chrony.txt
ansible-playbook playbooks/disable_ipv6.yml --tags deploy  > ../reports/playbooks/disable_ipv6.txt
ansible-playbook playbooks/routing.yml --tags deploy  > ../reports/playbooks/routing.txt
ansible-playbook playbooks/exit_node.yml --tags deploy > ../reports/playbooks/exit_node.txt
ansible-playbook playbooks/intermediateTestOfNetworkConnectivity.yml  > ../reports/playbooks/intermediateTestOfNetworkConnectivity.txt

ansible-playbook playbooks/postgresql.yml --tags deploy > ../reports/playbooks/postgresql.txt
ansible-playbook playbooks/intermediateTestOfDbAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfDbAvailabilityFromExternalHost.txt

ansible-playbook playbooks/django.yml --tags deploy > ../reports/playbooks/django.txt
ansible-playbook playbooks/intermediateTestOfAppAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfAppAvailabilityFromExternalHost.txt

ansible-playbook playbooks/nginx.yml --tags deploy_django > ../reports/playbooks/nginx.txt
ansible-playbook playbooks/intermediateTestOfWebAvailabilityFromExternalHost.yml> ./reports/playbooks/intermediateTestOfWebAvailabilityFromExternalHost.txt

ansible-playbook playbooks/rsyslog_server.yml --tags deploy > ../reports/playbooks/rsyslog_server.txt
ansible-playbook playbooks/rsyslog_clients.yml --tags clients > ../reports/playbooks/rsyslog_clients.txt

```

