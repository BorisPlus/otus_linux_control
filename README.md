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



<details><summary>см. импорт Vagrant хостов в Ansible inventory</summary>

```python3
from vagranttoansible import vagranttoansible

# python3 v2a.py -o ../ansible/inventories/hosts
# vagranttoansible.DEFAULT_LINE_FORMAT = f"{vagranttoansible.DEFAULT_LINE_FORMAT} ansible_ssh_transfer_method=scp ansible_python_interpreter=/usr/bin/python3"
vagranttoansible.DEFAULT_LINE_FORMAT = f"{vagranttoansible.DEFAULT_LINE_FORMAT} ansible_ssh_transfer_method=scp ansible_python_interpreter=/usr/bin/python"
# vagranttoansible.DEFAULT_LINE_FORMAT = f"{vagranttoansible.DEFAULT_LINE_FORMAT} ansible_ssh_transfer_method=scp"
# vagranttoansible.DEFAULT_LINE_FORMAT = "[{host}]\n" \
#                                        "{host} " \
#                                        "ansible_user={user} " \
#                                        "ansible_host={ssh_hostname} " \
#                                        "ansible_port={ssh_port} " \
#                                        "ansible_private_key_file={private_file}"
vagranttoansible.cli()

```

</details>

```shell
python3 v2a.py -o ../ansible/inventories/hosts
```

Для его использования можно сделать

```shell
pip3 install -r requirements.txt
```

Применил хранение настроек в переменной, для однотипной задачи прописывания шлюзов на интерфейсах


<details><summary>см. Параметры шлюзов</summary>

```properties
networks_hosts:
  internetRouter:
    default_gateway:
      interface: eth0
      gw: ''
    routes:
      - { interface: eth1, nw: 192.168.0.0/16, via: 192.168.255.2 }
    forwarding: 1
  centralRouter:
    default_gateway:
      interface: eth1
      gw: 192.168.255.1
    routes:
      - { interface: eth3, nw: 192.168.2.0/24, via: 192.168.0.34 }
      - { interface: eth3, nw: 192.168.1.0/24, via: 192.168.0.35 }
    forwarding: 1
  webServer:
    default_gateway:
      interface: eth1
      gw: 192.168.0.1
    routes: []
  appRouter:
    default_gateway:
      interface: eth1
      gw: 192.168.0.33
    routes: []
    forwarding: 1
  appServer:
    default_gateway:
      interface: eth1
      gw: 192.168.2.193
    routes: []
  dbRouter:
    default_gateway:
      interface: eth1
      gw: 192.168.0.33
    routes: []
    forwarding: 1
  dbServer:
    default_gateway:
      interface: eth1
      gw: 192.168.1.193
    routes: []
  dbBackupServer:
    default_gateway:
      interface: eth1
      gw: 192.168.1.193
    routes: []
  monitoringServer:
    default_gateway:
      interface: eth1
      gw: 192.168.0.65
    routes: []

```

</details>

Задача, использующая специальную ansible-переменную `inventory_hostname`, "знающую" для какого инвентори-хоста работает в данный момент плейбук.


<details><summary>см. Задача</summary>

```properties
---
- name: /etc/sysconfig/network | "NOZEROCONF=yes" | I don't want 169.254.0.0/16 network at default
  lineinfile:
    insertafter: EOF
    dest: /etc/sysconfig/network
    line: "NOZEROCONF=yes"
  ignore_errors: false
  tags:
    - deploy

- name: /etc/sysconfig/network-scripts/route-* | delete route files
  file:
    path: /etc/sysconfig/network-scripts/route-*
    state: absent
    force: yes
  ignore_errors: yes
  tags:
    - deploy

- name: /etc/sysconfig/network-scripts/route-* | create needed route files
  file:
    path: /etc/sysconfig/network-scripts/route-{{ item['interface'] }}
    state: touch
    owner: root
    group: root
    mode: 0644
  loop: "{{ networks_hosts[inventory_hostname]['routes'] }}"
  tags:
    - deploy

- name: /etc/sysconfig/network-scripts/route-* | set route
  lineinfile:
    insertafter: EOF
    dest: /etc/sysconfig/network-scripts/route-{{ item['interface'] }}
    line: "{{ item.nw }} via {{ item.via }}"
  loop: "{{ networks_hosts[inventory_hostname]['routes'] }}"
  notify:
    - systemctl-restart-network
  tags:
    - deploy

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | content
  command: /bin/cat /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
  when: (default_gateway['interface'] is defined and default_gateway['interface'] != '') and not (default_gateway['interface'] == None)
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  register: ifcfg_ethX_content
  tags:
    - ifcfg-ethX-default_gateway
    - deploy

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | GATEWAY=<ip> | set up if did not set
  lineinfile:
    insertafter: EOF
    dest: /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
    line: "GATEWAY={{ default_gateway['gw'] }}"
  when: ( default_gateway['gw'] is defined and default_gateway['gw'] != '') and not ( ifcfg_ethX_content is search("GATEWAY=") )
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  notify:
    - systemctl-restart-network
  tags:
    - deploy
    - ifcfg-ethX-default-gw

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | GATEWAY=<ip> | replace
  replace:
    path:  /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
    regexp: 'GATEWAY=[\w\.]*'
    replace: "GATEWAY={{ default_gateway['gw'] }}"
  when: (default_gateway['gw'] is defined and default_gateway['gw'] != '') and ( ifcfg_ethX_content is search("GATEWAY=") )
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  notify:
    - systemctl-restart-network
  tags:
    - deploy
    - ifcfg-ethX-default-gw

- name: /etc/sysconfig/network-scripts/ifcfg-eth0 | content
  command: /bin/cat /etc/sysconfig/network-scripts/ifcfg-eth0
  register: ifcfg_eth0_content
  tags:
    - deploy

- name: /etc/sysconfig/network-scripts/ifcfg-eth0 | DEFROUTE=no | if did not set
  lineinfile:
    insertafter: EOF
    dest: /etc/sysconfig/network-scripts/ifcfg-eth0
    line: "DEFROUTE=no"
  when: not ifcfg_eth0_content is search("DEFROUTE=")
  notify:
    - systemctl-restart-network
  tags:
    - deploy
    - ifcfg-eth0-default-route-off

- name: /etc/sysconfig/network-scripts/ifcfg-eth0 | DEFROUTE=no | if "yes"
  replace:
    path: /etc/sysconfig/network-scripts/ifcfg-eth0
    regexp: 'DEFROUTE=yes'
    replace: 'DEFROUTE=no'
  notify:
    - systemctl-restart-network
  tags:
    - deploy
    - ifcfg-eth0-default-route-off

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | content
  command: /bin/cat /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
  when: (default_gateway['interface'] is defined and default_gateway['interface'] != '') and not (default_gateway['interface'] == None)
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  register: ifcfg_ethX_content
  tags:
    - deploy
    - ifcfg-ethX-defroute

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | DEFROUTE=yes  | if did not set
  lineinfile:
    insertafter: EOF
    dest: /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
    line: "DEFROUTE=yes"
  when: (default_gateway['interface'] is defined and default_gateway['interface'] != '') and not ifcfg_ethX_content is search("DEFROUTE=")
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  notify:
    - systemctl-restart-network
  tags:
    - ifcfg-ethX-defroute
    - deploy

- name: /etc/sysconfig/network-scripts/ifcfg-ethX | DEFROUTE=yes | if "no"
  replace:
    path: /etc/sysconfig/network-scripts/ifcfg-{{ default_gateway['interface'] }}
    regexp: 'DEFROUTE=no'
    replace: 'DEFROUTE=yes'
  when: (default_gateway['interface'] is defined and default_gateway['interface'] != '') and ifcfg_ethX_content is search("DEFROUTE=no")
  vars:
    default_gateway: "{{ networks_hosts[inventory_hostname]['default_gateway'] }}"
  notify:
    - systemctl-restart-network
  tags:
    - ifcfg-ethX-defroute
    - deploy

- name: /etc/sysctl.conf | content
  command: /bin/cat /etc/sysctl.conf
  register: sysctl_conf_content
  tags:
    - forwarding-set-up
    - deploy

- name: /etc/sysctl.conf | forwarding set up | if does not yet
  lineinfile:
    insertafter: EOF
    dest: /etc/sysctl.conf
    line: "net.ipv4.conf.all.forwarding={{ lets.forwarding }}"
  when: (lets.forwarding is defined and (lets.forwarding == 0 or lets.forwarding == 1)) and not sysctl_conf_content is search("net.ipv4.conf.all.forwarding")
  vars:
    lets: "{{ networks_hosts[inventory_hostname] }}"
  notify:
    - systemctl-restart-network
  tags:
    - forwarding-set-up
    - deploy

- name: /etc/sysctl.conf | forwarding set up | if it was early
  replace:
    path: /etc/sysctl.conf
    regexp: 'net\.ipv4\.conf\.all\.forwarding.*'
    replace: 'net.ipv4.conf.all.forwarding={{ lets.forwarding }}'
  when: (lets.forwarding is defined and (lets.forwarding == 0 or lets.forwarding == 1)) and sysctl_conf_content is search("net.ipv4.conf.all.forwarding")
  vars:
    lets: "{{ networks_hosts[inventory_hostname] }}"
  notify:
    - systemctl-restart-network
  tags:
    - forwarding-set-up
    - deploy

- name: /etc/sysctl.conf | ip_forward set up | if does not yet
  lineinfile:
    insertafter: EOF
    dest: /etc/sysctl.conf
    line: "net.ipv4.ip_forward={{ lets.ip_forward }}"
  when: (lets.ip_forward is defined and (lets.ip_forward == 0 or lets.ip_forward == 1)) and not sysctl_conf_content is search("net.ipv4.ip_forward")
  vars:
    lets: "{{ networks_hosts[inventory_hostname] }}"
  notify:
    - systemctl-restart-network
  tags:
    - ip_forward-set-up
    - deploy

- name: /etc/sysctl.conf | ip_forward set up | if it was early
  replace:
    path: /etc/sysctl.conf
    regexp: 'net\.ipv4\.ip_forward.*'
    replace: 'net.ipv4.ip_forward={{ lets.ip_forward }}'
  when: (lets.ip_forward is defined and (lets.ip_forward == 0 or lets.ip_forward == 1)) and sysctl_conf_content is search("net.ipv4.ip_forward")
  vars:
    lets: "{{ networks_hosts[inventory_hostname] }}"
  notify:
    - systemctl-restart-network
  tags:
    - ip_forward-set-up
    - deploy

- name: network manager war | chattr -i /etc/resolv.conf
  file:
    path: /etc/resolv.conf
    attributes: '-i'
  tags:
    - deploy

- name: copy resolv.conf to the servers
  copy:
    src: ../files/resolv.conf
    dest: /etc/resolv.conf
    owner: root
    group: root
    mode: 0644
  tags:
    - deploy

- name: network manager war | chattr +i /etc/resolv.conf
  file:
    path: /etc/resolv.conf
    attributes: '+i'
  tags:
    - deploy
```

</details>

Аналогично в рамках демонстрации работоспособности применил `loop` и `inventory_hostname` 


<details><summary>см. Задача</summary>

```properties
---
- name: yum install traceroute
  yum:
    name: traceroute
    state: latest
  become: yes
  tags:
    - yum-install
    - traceroute
    - deploy

- name: test | demostration
  debug:
    msg: "I am {{ inventory_hostname }}, I get self name from `inventory_hostname` ansible-variable."

- name: test | traceroute foreign host
  shell: traceroute {{ item }}
  loop: "{{ to }}"
  ignore_errors: true
  register: traceroute_content
  tags:
    - clear
    - naming
    - deploy

- name: test | delete preverios report
  local_action:
    module: file
    path: "../../reports/tests/routing/{{ inventory_hostname }} - {{ item['cmd'] }}.txt"
    force: yes
    state: absent
  loop: "{{ traceroute_content['results'] }}"
  tags:
    - traceroute
    - clear
    - deploy

- name: test | strore trace result
  local_action:
    module: lineinfile
    dest: "../../reports/tests/routing/{{ inventory_hostname }} - {{ item['cmd'] }}.txt"
    line: "{{ item['stdout'] }}"
    create: yes
  loop: "{{ traceroute_content['results'] }}"
  changed_when: true
  tags:
    - traceroute
    - deploy

- name: yum uninstall traceroute
  become: yes
  yum:
    name: traceroute
    state: absent
  tags:
    - clear
    - traceroute
    - deploy
```

</details>

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

На стенде трафик маршрутизируется исключительно в IPv4. У меня возникали проблемы с доступом YUM к репозиториям, я отключил IPv6 совсем и пошло быстрее.

```shell
ansible-playbook playbooks/disable_ipv6.yml  > ../reports/playbooks/disable_ipv6.txt
```


<details><summary>см. playbooks/disable_ipv6.yml</summary>

```text

PLAY [Playbook of ip v6 disable] ***********************************************

TASK [Gathering Facts] *********************************************************
ok: [internetRouter]
ok: [webServer]
ok: [appRouter]
ok: [appServer]
ok: [centralRouter]
ok: [dbServer]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf] *****************
changed: [webServer]
changed: [appRouter]
changed: [appServer]
changed: [centralRouter]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | content] *******
changed: [centralRouter]
changed: [webServer]
changed: [appServer]
changed: [appRouter]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [dbServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.all.disable_ipv6=1 | if does not yet] ***
changed: [internetRouter]
changed: [appServer]
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [dbServer]
changed: [dbRouter]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.default.disable_ipv6=1 | if does not yet] ***
changed: [internetRouter]
changed: [centralRouter]
changed: [appRouter]
changed: [appServer]
changed: [webServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.default.disable_ipv6 = 1 | if it was early] ***
ok: [appRouter]
ok: [centralRouter]
ok: [webServer]
ok: [internetRouter]
ok: [appServer]
ok: [dbServer]
ok: [monitoringServer]
ok: [dbBackupServer]
ok: [dbRouter]

TASK [../roles/disable_ipv6 : /etc/yum.conf | content] *************************
changed: [centralRouter]
changed: [internetRouter]
changed: [appRouter]
changed: [webServer]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/yum.conf | ip_resolve=4 | if does not yet] ***
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [internetRouter]
changed: [appServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [dbServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/yum.conf | ip_resolve=4 | if it was early] ***
ok: [webServer]
ok: [internetRouter]
ok: [appServer]
ok: [centralRouter]
ok: [appRouter]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [dbServer]
ok: [monitoringServer]

RUNNING HANDLER [../roles/disable_ipv6 : sysctl-p] *****************************
changed: [webServer]
changed: [appServer]
changed: [centralRouter]
changed: [appRouter]
changed: [internetRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbRouter]
changed: [dbBackupServer]

RUNNING HANDLER [../roles/disable_ipv6 : systemctl-restart-network] ************
changed: [internetRouter]
changed: [appServer]
changed: [webServer]
changed: [dbServer]
changed: [monitoringServer]
changed: [appRouter]
changed: [centralRouter]
changed: [dbRouter]
changed: [dbBackupServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
appServer                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centralRouter              : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbBackupServer             : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbServer                   : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
internetRouter             : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
monitoringServer           : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webServer                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

#### Общее. Настройка сети

Одной командой настраиваем сеть инфраструктуры 

```shell
ansible-playbook playbooks/chrony.yml --tags deploy > ../reports/playbooks/chrony.txt
```


<details><summary>см. playbooks/chrony.yml</summary>

```text

PLAY [Playbook of time chrony] *************************************************

TASK [Gathering Facts] *********************************************************
ok: [centralRouter]
ok: [appServer]
ok: [appRouter]
ok: [internetRouter]
ok: [webServer]
ok: [monitoringServer]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [dbServer]

TASK [../roles/chrony : Synchronize datetime | Install chrony] *****************
ok: [appServer]
ok: [webServer]
ok: [internetRouter]
ok: [appRouter]
ok: [centralRouter]
ok: [monitoringServer]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [dbServer]

TASK [../roles/chrony : Synchronize datetime | Turn on chronyd] ****************
ok: [internetRouter]
ok: [webServer]
ok: [centralRouter]
ok: [appRouter]
ok: [appServer]
ok: [dbServer]
ok: [dbRouter]
ok: [monitoringServer]
ok: [dbBackupServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
appServer                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centralRouter              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbBackupServer             : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbServer                   : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
internetRouter             : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
monitoringServer           : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webServer                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

```shell
ansible-playbook playbooks/disable_ipv6.yml --tags deploy  > ../reports/playbooks/disable_ipv6.txt
```


<details><summary>см. playbooks/disable_ipv6.yml</summary>

```text

PLAY [Playbook of ip v6 disable] ***********************************************

TASK [Gathering Facts] *********************************************************
ok: [internetRouter]
ok: [webServer]
ok: [appRouter]
ok: [appServer]
ok: [centralRouter]
ok: [dbServer]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf] *****************
changed: [webServer]
changed: [appRouter]
changed: [appServer]
changed: [centralRouter]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | content] *******
changed: [centralRouter]
changed: [webServer]
changed: [appServer]
changed: [appRouter]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [dbServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.all.disable_ipv6=1 | if does not yet] ***
changed: [internetRouter]
changed: [appServer]
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [dbServer]
changed: [dbRouter]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.default.disable_ipv6=1 | if does not yet] ***
changed: [internetRouter]
changed: [centralRouter]
changed: [appRouter]
changed: [appServer]
changed: [webServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/sysctl.d/disable-ipv6.conf | net.ipv6.conf.default.disable_ipv6 = 1 | if it was early] ***
ok: [appRouter]
ok: [centralRouter]
ok: [webServer]
ok: [internetRouter]
ok: [appServer]
ok: [dbServer]
ok: [monitoringServer]
ok: [dbBackupServer]
ok: [dbRouter]

TASK [../roles/disable_ipv6 : /etc/yum.conf | content] *************************
changed: [centralRouter]
changed: [internetRouter]
changed: [appRouter]
changed: [webServer]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/yum.conf | ip_resolve=4 | if does not yet] ***
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [internetRouter]
changed: [appServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [dbServer]
changed: [monitoringServer]

TASK [../roles/disable_ipv6 : /etc/yum.conf | ip_resolve=4 | if it was early] ***
ok: [webServer]
ok: [internetRouter]
ok: [appServer]
ok: [centralRouter]
ok: [appRouter]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [dbServer]
ok: [monitoringServer]

RUNNING HANDLER [../roles/disable_ipv6 : sysctl-p] *****************************
changed: [webServer]
changed: [appServer]
changed: [centralRouter]
changed: [appRouter]
changed: [internetRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbRouter]
changed: [dbBackupServer]

RUNNING HANDLER [../roles/disable_ipv6 : systemctl-restart-network] ************
changed: [internetRouter]
changed: [appServer]
changed: [webServer]
changed: [dbServer]
changed: [monitoringServer]
changed: [appRouter]
changed: [centralRouter]
changed: [dbRouter]
changed: [dbBackupServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
appServer                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centralRouter              : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbBackupServer             : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbServer                   : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
internetRouter             : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
monitoringServer           : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webServer                  : ok=11   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

```shell
ansible-playbook playbooks/routing.yml --tags deploy  > ../reports/playbooks/routing.txt
```


<details><summary>см. playbooks/routing.yml</summary>

```text

PLAY [Playbook of eth[X] configure] ********************************************

TASK [Gathering Facts] *********************************************************
ok: [centralRouter]
ok: [internetRouter]
ok: [webServer]
ok: [appServer]
ok: [appRouter]
ok: [dbBackupServer]
ok: [dbServer]
ok: [dbRouter]
ok: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network | "NOZEROCONF=yes" | I don't want 169.254.0.0/16 network at default] ***
changed: [centralRouter]
changed: [appRouter]
changed: [webServer]
changed: [appServer]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/route-* | delete route files] ***
ok: [appServer]
ok: [webServer]
ok: [internetRouter]
ok: [centralRouter]
ok: [appRouter]
ok: [dbRouter]
ok: [dbBackupServer]
ok: [dbServer]
ok: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/route-* | create needed route files] ***
changed: [centralRouter] => (item={'interface': 'eth3', 'nw': '192.168.2.0/24', 'via': '192.168.0.34'})
changed: [internetRouter] => (item={'interface': 'eth1', 'nw': '192.168.0.0/16', 'via': '192.168.255.2'})
changed: [centralRouter] => (item={'interface': 'eth3', 'nw': '192.168.1.0/24', 'via': '192.168.0.35'})

TASK [../roles/routing : /etc/sysconfig/network-scripts/route-* | set route] ***
changed: [internetRouter] => (item={'interface': 'eth1', 'nw': '192.168.0.0/16', 'via': '192.168.255.2'})
changed: [centralRouter] => (item={'interface': 'eth3', 'nw': '192.168.2.0/24', 'via': '192.168.0.34'})
changed: [centralRouter] => (item={'interface': 'eth3', 'nw': '192.168.1.0/24', 'via': '192.168.0.35'})

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | content] ***
changed: [appServer]
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | GATEWAY=<ip> | set up if did not set] ***
skipping: [internetRouter]
changed: [appServer]
changed: [appRouter]
changed: [centralRouter]
changed: [dbRouter]
changed: [webServer]
changed: [dbBackupServer]
changed: [dbServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | GATEWAY=<ip> | replace] ***
skipping: [internetRouter]
skipping: [centralRouter]
skipping: [webServer]
skipping: [appRouter]
skipping: [appServer]
skipping: [dbRouter]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-eth0 | content] ***
changed: [centralRouter]
changed: [webServer]
changed: [appRouter]
changed: [internetRouter]
changed: [appServer]
changed: [dbRouter]
changed: [dbServer]
changed: [monitoringServer]
changed: [dbBackupServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-eth0 | DEFROUTE=no | if did not set] ***
changed: [centralRouter]
changed: [webServer]
changed: [appRouter]
changed: [appServer]
changed: [internetRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-eth0 | DEFROUTE=no | if "yes"] ***
ok: [centralRouter]
ok: [internetRouter]
ok: [webServer]
ok: [appRouter]
ok: [appServer]
ok: [dbRouter]
ok: [dbServer]
ok: [monitoringServer]
ok: [dbBackupServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | content] ***
changed: [internetRouter]
changed: [appRouter]
changed: [centralRouter]
changed: [webServer]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | DEFROUTE=yes  | if did not set] ***
skipping: [internetRouter]
changed: [appRouter]
changed: [webServer]
changed: [dbRouter]
changed: [appServer]
changed: [centralRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysconfig/network-scripts/ifcfg-ethX | DEFROUTE=yes | if "no"] ***
skipping: [centralRouter]
skipping: [webServer]
skipping: [appRouter]
skipping: [appServer]
skipping: [dbRouter]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]
changed: [internetRouter]

TASK [../roles/routing : /etc/sysctl.conf | content] ***************************
changed: [internetRouter]
changed: [centralRouter]
changed: [appRouter]
changed: [webServer]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : /etc/sysctl.conf | forwarding set up | if does not yet] ***
skipping: [webServer]
skipping: [appServer]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]
changed: [internetRouter]
changed: [appRouter]
changed: [centralRouter]
changed: [dbRouter]

TASK [../roles/routing : /etc/sysctl.conf | forwarding set up | if it was early] ***
skipping: [internetRouter]
skipping: [centralRouter]
skipping: [webServer]
skipping: [appRouter]
skipping: [appServer]
skipping: [dbRouter]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]

TASK [../roles/routing : /etc/sysctl.conf | ip_forward set up | if does not yet] ***
skipping: [internetRouter]
skipping: [centralRouter]
skipping: [webServer]
skipping: [appRouter]
skipping: [appServer]
skipping: [dbRouter]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]

TASK [../roles/routing : /etc/sysctl.conf | ip_forward set up | if it was early] ***
skipping: [internetRouter]
skipping: [centralRouter]
skipping: [webServer]
skipping: [appRouter]
skipping: [appServer]
skipping: [dbRouter]
skipping: [dbServer]
skipping: [dbBackupServer]
skipping: [monitoringServer]

TASK [../roles/routing : network manager war | chattr -i /etc/resolv.conf] *****
changed: [internetRouter]
changed: [appRouter]
changed: [centralRouter]
changed: [appServer]
changed: [webServer]
changed: [dbRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : copy resolv.conf to the servers] **********************
changed: [appServer]
changed: [internetRouter]
changed: [webServer]
changed: [centralRouter]
changed: [appRouter]
changed: [dbRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [monitoringServer]

TASK [../roles/routing : network manager war | chattr +i /etc/resolv.conf] *****
changed: [centralRouter]
changed: [webServer]
changed: [internetRouter]
changed: [appServer]
changed: [appRouter]
changed: [dbServer]
changed: [dbBackupServer]
changed: [dbRouter]
changed: [monitoringServer]

RUNNING HANDLER [../roles/routing : systemctl-restart-network] *****************
changed: [internetRouter]
changed: [appServer]
changed: [webServer]
changed: [dbRouter]
changed: [centralRouter]
changed: [dbBackupServer]
changed: [appRouter]
changed: [monitoringServer]
changed: [dbServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=16   changed=13   unreachable=0    failed=0    skipped=7    rescued=0    ignored=0   
appServer                  : ok=15   changed=12   unreachable=0    failed=0    skipped=8    rescued=0    ignored=0   
centralRouter              : ok=18   changed=15   unreachable=0    failed=0    skipped=5    rescued=0    ignored=0   
dbBackupServer             : ok=15   changed=12   unreachable=0    failed=0    skipped=8    rescued=0    ignored=0   
dbRouter                   : ok=16   changed=13   unreachable=0    failed=0    skipped=7    rescued=0    ignored=0   
dbServer                   : ok=15   changed=12   unreachable=0    failed=0    skipped=8    rescued=0    ignored=0   
internetRouter             : ok=17   changed=14   unreachable=0    failed=0    skipped=6    rescued=0    ignored=0   
monitoringServer           : ok=15   changed=12   unreachable=0    failed=0    skipped=8    rescued=0    ignored=0   
webServer                  : ok=15   changed=12   unreachable=0    failed=0    skipped=8    rescued=0    ignored=0   


```

</details>

Доступ вовне идет через один узел, которому нужно об этом "напомнить"

```shell
ansible-playbook playbooks/exit_node.yml --tags deploy > ../reports/playbooks/exit_node.txt
```


<details><summary>см. playbooks/exit_node.yml</summary>

```text

PLAY [Playbook of exit-node configure] *****************************************

TASK [Gathering Facts] *********************************************************
ok: [internetRouter]

TASK [../roles/exit_node : iptables configure] *********************************
changed: [internetRouter] => (item=iptables -F)
changed: [internetRouter] => (item=iptables -X)
changed: [internetRouter] => (item=iptables -F -t nat)
changed: [internetRouter] => (item=iptables -P FORWARD ACCEPT)
changed: [internetRouter] => (item=iptables -P OUTPUT ACCEPT)
changed: [internetRouter] => (item=iptables -P INPUT ACCEPT)
changed: [internetRouter] => (item=iptables -t nat -A POSTROUTING -t nat -o eth0 -j MASQUERADE)
changed: [internetRouter] => (item=iptables -A FORWARD -i eth0 -o eth1 -p tcp --syn --dport 80 -m conntrack --ctstate NEW -j ACCEPT)
changed: [internetRouter] => (item=iptables -A FORWARD -i eth0 -o eth1 -p tcp --syn --dport 443 -m conntrack --ctstate NEW -j ACCEPT)
changed: [internetRouter] => (item=iptables -A FORWARD -i eth0 -o eth1 -p tcp --syn --dport 8000 -m conntrack --ctstate NEW -j ACCEPT)
changed: [internetRouter] => (item=iptables -A FORWARD -i eth0 -o eth1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT)
changed: [internetRouter] => (item=iptables -A FORWARD -i eth1 -o eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT)
changed: [internetRouter] => (item=iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.0.2)
changed: [internetRouter] => (item=iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 192.168.0.2)
changed: [internetRouter] => (item=iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8000 -j DNAT --to-destination 192.168.2.194)
changed: [internetRouter] => (item=iptables -t nat -A POSTROUTING -o eth1 -p tcp --dport 80 -d 192.168.0.2 -j SNAT --to-source 192.168.255.1)
changed: [internetRouter] => (item=iptables -t nat -A POSTROUTING -o eth1 -p tcp --dport 443 -d 192.168.0.2 -j SNAT --to-source 192.168.255.1)
changed: [internetRouter] => (item=iptables -t nat -A POSTROUTING -o eth1 -p tcp --dport 8000 -d 192.168.2.194 -j SNAT --to-source 192.168.255.1)

TASK [../roles/exit_node : yum install iptables-services] **********************
changed: [internetRouter] => (item=iptables-services)

TASK [../roles/exit_node : systemctl enable iptables] **************************
changed: [internetRouter]

TASK [../roles/exit_node : service iptables save] ******************************
changed: [internetRouter]

TASK [../roles/exit_node : iptables --table <table> --list] ********************
changed: [internetRouter] => (item=filter)
changed: [internetRouter] => (item=nat)
changed: [internetRouter] => (item=mangle)
changed: [internetRouter] => (item=raw)
changed: [internetRouter] => (item=security)

TASK [../roles/exit_node : strore | iptables --table <table> --list] ***********
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:21:25.408402', 'stdout': 'Chain INPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain FORWARD (policy ACCEPT)\ntarget     prot opt source               destination         \nACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http flags:FIN,SYN,RST,ACK/SYN ctstate NEW\nACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https flags:FIN,SYN,RST,ACK/SYN ctstate NEW\nACCEPT     tcp  --  anywhere             anywhere             tcp dpt:irdmi flags:FIN,SYN,RST,ACK/SYN ctstate NEW\nACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED\nACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED\n\nChain OUTPUT (policy ACCEPT)\ntarget     prot opt source               destination         ', 'cmd': 'iptables --table filter --list', 'rc': 0, 'start': '2021-11-30 13:21:25.380652', 'stderr': '', 'delta': '0:00:00.027750', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'iptables --table filter --list', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['Chain INPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain FORWARD (policy ACCEPT)', 'target     prot opt source               destination         ', 'ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http flags:FIN,SYN,RST,ACK/SYN ctstate NEW', 'ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https flags:FIN,SYN,RST,ACK/SYN ctstate NEW', 'ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:irdmi flags:FIN,SYN,RST,ACK/SYN ctstate NEW', 'ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED', 'ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED', '', 'Chain OUTPUT (policy ACCEPT)', 'target     prot opt source               destination         '], 'stderr_lines': [], 'failed': False, 'item': 'filter', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:21:26.237482', 'stdout': 'Chain PREROUTING (policy ACCEPT)\ntarget     prot opt source               destination         \nDNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:192.168.0.2\nDNAT       tcp  --  anywhere             anywhere             tcp dpt:https to:192.168.0.2\nDNAT       tcp  --  anywhere             anywhere             tcp dpt:irdmi to:192.168.2.194\n\nChain INPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain OUTPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain POSTROUTING (policy ACCEPT)\ntarget     prot opt source               destination         \nMASQUERADE  all  --  anywhere             anywhere            \nSNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:http to:192.168.255.1\nSNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:https to:192.168.255.1\nSNAT       tcp  --  anywhere             192.168.2.194        tcp dpt:irdmi to:192.168.255.1', 'cmd': 'iptables --table nat --list', 'rc': 0, 'start': '2021-11-30 13:21:26.006123', 'stderr': '', 'delta': '0:00:00.231359', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'iptables --table nat --list', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['Chain PREROUTING (policy ACCEPT)', 'target     prot opt source               destination         ', 'DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:192.168.0.2', 'DNAT       tcp  --  anywhere             anywhere             tcp dpt:https to:192.168.0.2', 'DNAT       tcp  --  anywhere             anywhere             tcp dpt:irdmi to:192.168.2.194', '', 'Chain INPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain OUTPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain POSTROUTING (policy ACCEPT)', 'target     prot opt source               destination         ', 'MASQUERADE  all  --  anywhere             anywhere            ', 'SNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:http to:192.168.255.1', 'SNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:https to:192.168.255.1', 'SNAT       tcp  --  anywhere             192.168.2.194        tcp dpt:irdmi to:192.168.255.1'], 'stderr_lines': [], 'failed': False, 'item': 'nat', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:21:26.865836', 'stdout': 'Chain PREROUTING (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain INPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain FORWARD (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain OUTPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain POSTROUTING (policy ACCEPT)\ntarget     prot opt source               destination         ', 'cmd': 'iptables --table mangle --list', 'rc': 0, 'start': '2021-11-30 13:21:26.838065', 'stderr': '', 'delta': '0:00:00.027771', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'iptables --table mangle --list', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['Chain PREROUTING (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain INPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain FORWARD (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain OUTPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain POSTROUTING (policy ACCEPT)', 'target     prot opt source               destination         '], 'stderr_lines': [], 'failed': False, 'item': 'mangle', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:21:27.495355', 'stdout': 'Chain PREROUTING (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain OUTPUT (policy ACCEPT)\ntarget     prot opt source               destination         ', 'cmd': 'iptables --table raw --list', 'rc': 0, 'start': '2021-11-30 13:21:27.475845', 'stderr': '', 'delta': '0:00:00.019510', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'iptables --table raw --list', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['Chain PREROUTING (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain OUTPUT (policy ACCEPT)', 'target     prot opt source               destination         '], 'stderr_lines': [], 'failed': False, 'item': 'raw', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:21:28.097942', 'stdout': 'Chain INPUT (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain FORWARD (policy ACCEPT)\ntarget     prot opt source               destination         \n\nChain OUTPUT (policy ACCEPT)\ntarget     prot opt source               destination         ', 'cmd': 'iptables --table security --list', 'rc': 0, 'start': '2021-11-30 13:21:28.081858', 'stderr': '', 'delta': '0:00:00.016084', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'iptables --table security --list', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['Chain INPUT (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain FORWARD (policy ACCEPT)', 'target     prot opt source               destination         ', '', 'Chain OUTPUT (policy ACCEPT)', 'target     prot opt source               destination         '], 'stderr_lines': [], 'failed': False, 'item': 'security', 'ansible_loop_var': 'item'})

PLAY RECAP *********************************************************************
internetRouter             : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

```shell
ansible-playbook playbooks/intermediateTestOfNetworkConnectivity.yml  > ../reports/playbooks/intermediateTestOfNetworkConnectivity.txt
```


<details><summary>см. playbooks/intermediateTestOfNetworkConnectivity.yml</summary>

```text

PLAY [Playbook of test network connectivity] ***********************************

TASK [Gathering Facts] *********************************************************
ok: [webServer]
ok: [centralRouter]
ok: [appRouter]
ok: [appServer]
ok: [internetRouter]
ok: [dbRouter]
ok: [dbServer]
ok: [monitoringServer]
ok: [dbBackupServer]

TASK [../roles/intermediateTestOfNetworkConnectivity : yum install traceroute] ***
changed: [internetRouter]
changed: [centralRouter]
changed: [appServer]
changed: [webServer]
changed: [appRouter]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]
changed: [dbServer]

TASK [../roles/intermediateTestOfNetworkConnectivity : test | demostration] ****
ok: [internetRouter] => {
    "msg": "I am internetRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [centralRouter] => {
    "msg": "I am centralRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [webServer] => {
    "msg": "I am webServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [appRouter] => {
    "msg": "I am appRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [appServer] => {
    "msg": "I am appServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbRouter] => {
    "msg": "I am dbRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbServer] => {
    "msg": "I am dbServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbBackupServer] => {
    "msg": "I am dbBackupServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [monitoringServer] => {
    "msg": "I am monitoringServer, I get self name from `inventory_hostname` ansible-variable."
}

TASK [../roles/intermediateTestOfNetworkConnectivity : test | traceroute foreign host] ***
changed: [internetRouter] => (item=8.8.8.8)
changed: [internetRouter] => (item=192.168.255.1)
changed: [internetRouter] => (item=192.168.255.2)
changed: [centralRouter] => (item=8.8.8.8)
changed: [appServer] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.255.1)
changed: [appServer] => (item=192.168.255.1)
changed: [centralRouter] => (item=192.168.255.2)
changed: [appServer] => (item=192.168.255.2)
changed: [webServer] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.0.2)
changed: [centralRouter] => (item=192.168.2.194)
changed: [internetRouter] => (item=192.168.0.2)
changed: [centralRouter] => (item=192.168.1.194)
changed: [internetRouter] => (item=192.168.2.194)
changed: [centralRouter] => (item=192.168.1.195)
changed: [appRouter] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.0.66)
changed: [internetRouter] => (item=192.168.1.194)
changed: [appRouter] => (item=192.168.255.1)
changed: [appRouter] => (item=192.168.255.2)
changed: [appServer] => (item=192.168.0.2)
changed: [webServer] => (item=192.168.255.1)
changed: [appServer] => (item=192.168.2.194)
changed: [webServer] => (item=192.168.255.2)
changed: [webServer] => (item=192.168.0.2)
changed: [appServer] => (item=192.168.1.194)
changed: [appServer] => (item=192.168.1.195)
changed: [internetRouter] => (item=192.168.1.195)
changed: [internetRouter] => (item=192.168.0.66)
changed: [webServer] => (item=192.168.2.194)
changed: [webServer] => (item=192.168.1.194)
changed: [appServer] => (item=192.168.0.66)
changed: [webServer] => (item=192.168.1.195)
changed: [appRouter] => (item=192.168.0.2)
changed: [appRouter] => (item=192.168.2.194)
changed: [appRouter] => (item=192.168.1.194)
changed: [appRouter] => (item=192.168.1.195)
changed: [webServer] => (item=192.168.0.66)
changed: [dbRouter] => (item=8.8.8.8)
changed: [dbRouter] => (item=192.168.255.1)
changed: [appRouter] => (item=192.168.0.66)
changed: [dbRouter] => (item=192.168.255.2)
changed: [dbRouter] => (item=192.168.0.2)
changed: [dbRouter] => (item=192.168.2.194)
changed: [dbRouter] => (item=192.168.1.194)
changed: [dbRouter] => (item=192.168.1.195)
changed: [dbRouter] => (item=192.168.0.66)
changed: [dbBackupServer] => (item=8.8.8.8)
changed: [dbBackupServer] => (item=192.168.255.1)
changed: [dbBackupServer] => (item=192.168.255.2)
changed: [dbBackupServer] => (item=192.168.0.2)
changed: [dbBackupServer] => (item=192.168.2.194)
changed: [dbBackupServer] => (item=192.168.1.194)
changed: [dbBackupServer] => (item=192.168.1.195)
changed: [dbBackupServer] => (item=192.168.0.66)
changed: [monitoringServer] => (item=8.8.8.8)
changed: [dbServer] => (item=8.8.8.8)
changed: [monitoringServer] => (item=192.168.255.1)
changed: [dbServer] => (item=192.168.255.1)
changed: [monitoringServer] => (item=192.168.255.2)
changed: [dbServer] => (item=192.168.255.2)
changed: [monitoringServer] => (item=192.168.0.2)
changed: [dbServer] => (item=192.168.0.2)
changed: [monitoringServer] => (item=192.168.2.194)
changed: [monitoringServer] => (item=192.168.1.194)
changed: [dbServer] => (item=192.168.2.194)
changed: [dbServer] => (item=192.168.1.194)
changed: [dbServer] => (item=192.168.1.195)
changed: [dbServer] => (item=192.168.0.66)
changed: [monitoringServer] => (item=192.168.1.195)
changed: [monitoringServer] => (item=192.168.0.66)

TASK [../roles/intermediateTestOfNetworkConnectivity : test | delete preverios report] ***
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.463805', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms\n 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms\n11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms\n14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms\n15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms\n16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms\n17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  64.830 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.772193', 'stderr': '', 'delta': '0:00:24.691612', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms', ' 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms', '11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms', '14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms', '15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms', '16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms', '17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  64.830 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.715594', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms\n 2  * * *\n 3  * * *\n 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms\n 5  * * *\n 6  * * *\n 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms\n 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms\n 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms\n10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms\n11  * * *\n12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms\n13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms\n14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms\n15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms\n16  * 172.253.51.187 (172.253.51.187)  49.992 ms *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  dns.google (8.8.8.8)  65.551 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.714654', 'stderr': '', 'delta': '0:00:23.000940', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms', ' 2  * * *', ' 3  * * *', ' 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms', ' 5  * * *', ' 6  * * *', ' 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms', ' 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms', ' 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms', '10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms', '11  * * *', '12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms', '13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms', '14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms', '15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms', '16  * 172.253.51.187 (172.253.51.187)  49.992 ms *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  dns.google (8.8.8.8)  65.551 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.781617', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms\n 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms\n 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms\n12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms\n13  * * *\n14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms\n15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms\n16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms\n17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms\n18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.716127', 'stderr': '', 'delta': '0:00:23.065490', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms', ' 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms', ' 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms', '12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms', '13  * * *', '14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms', '15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms', '16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms', '17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms', '18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.188820', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms\n 2  * * *\n 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms\n 4  * * *\n 5  * * *\n 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms\n 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms\n 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms\n 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms\n10  * * *\n11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms\n12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms\n13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms\n14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms\n15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *\n16  * * *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.679519', 'stderr': '', 'delta': '0:00:19.509301', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms', ' 2  * * *', ' 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms', ' 4  * * *', ' 5  * * *', ' 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms', ' 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms', ' 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms', ' 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms', '10  * * *', '11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms', '12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms', '13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms', '14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms', '15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *', '16  * * *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.292177', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms\n 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms\n11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms\n12  * * *\n13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms\n14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms\n15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms\n16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.699399', 'stderr': '', 'delta': '0:00:27.592778', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms', ' 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms', '11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms', '12  * * *', '13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms', '14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms', '15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms', '16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.344975', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms\n 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:48.145386', 'stderr': '', 'delta': '0:00:05.199589', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms', ' 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.439494', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.318405', 'stderr': '', 'delta': '0:00:00.121089', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.853054', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:42.766411', 'stderr': '', 'delta': '0:00:00.086643', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.669218', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms\n 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms\n 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.375676', 'stderr': '', 'delta': '0:00:00.293542', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms', ' 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms', ' 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.098501', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms\n 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:50.926503', 'stderr': '', 'delta': '0:00:00.171998', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms', ' 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.089207', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:53.994077', 'stderr': '', 'delta': '0:00:00.095130', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:43.566372', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:43.440441', 'stderr': '', 'delta': '0:00:00.125931', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.147842', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.038789', 'stderr': '', 'delta': '0:00:00.109053', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.444010', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms\n 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.241063', 'stderr': '', 'delta': '0:00:00.202947', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms', ' 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:52.010000', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:51.910405', 'stderr': '', 'delta': '0:00:00.099595', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.770104', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:54.681551', 'stderr': '', 'delta': '0:00:00.088553', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.230515', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  * * *\n 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:44.134035', 'stderr': '', 'delta': '0:00:05.096480', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  * * *', ' 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.909389', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:47.808126', 'stderr': '', 'delta': '0:00:00.101263', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.286737', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.367 ms * *\n 2  192.168.0.33 (192.168.0.33)  2.995 ms * *\n 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:48.111492', 'stderr': '', 'delta': '0:00:05.175245', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.367 ms * *', ' 2  192.168.0.33 (192.168.0.33)  2.995 ms * *', ' 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.771173', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.327 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:52.572549', 'stderr': '', 'delta': '0:00:10.198624', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.327 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:00.590285', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.342 ms * *\n 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms\n 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:55.414887', 'stderr': '', 'delta': '0:00:05.175398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.342 ms * *', ' 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms', ' 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.069715', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms\n 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms\n 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:49.843490', 'stderr': '', 'delta': '0:00:00.226225', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms', ' 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms', ' 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:48.739323', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms\n 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:48.532562', 'stderr': '', 'delta': '0:00:00.206761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms', ' 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.048111', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:53.949070', 'stderr': '', 'delta': '0:00:00.099041', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.430540', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms\n 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms\n 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:01.153780', 'stderr': '', 'delta': '0:00:00.276760', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms', ' 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms', ' 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.067606', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms\n 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms\n 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:50.808896', 'stderr': '', 'delta': '0:00:00.258710', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms', ' 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms', ' 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:03.480218', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:03.365980', 'stderr': '', 'delta': '0:00:00.114238', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.470074', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms\n 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:49.316163', 'stderr': '', 'delta': '0:00:00.153911', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms', ' 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.965468', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms\n 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms\n 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms\n 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:54.658707', 'stderr': '', 'delta': '0:00:00.306761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms', ' 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms', ' 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms', ' 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.052522', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *\n 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms\n 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:51.893676', 'stderr': '', 'delta': '0:00:05.158846', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *', ' 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms', ' 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.283435', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms\n 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms\n 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:02.047439', 'stderr': '', 'delta': '0:00:00.235996', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms', ' 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms', ' 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:04.324858', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms\n 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms\n 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:04.075955', 'stderr': '', 'delta': '0:00:00.248903', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms', ' 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms', ' 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.188329', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms\n 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:50.035875', 'stderr': '', 'delta': '0:00:00.152454', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms', ' 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:55.867393', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms\n 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms\n 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms\n 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:55.572293', 'stderr': '', 'delta': '0:00:00.295100', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms', ' 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms', ' 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms', ' 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:07.968414', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.480 ms * *\n 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:02.860980', 'stderr': '', 'delta': '0:00:05.107434', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.480 ms * *', ' 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.798011', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms\n 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:57.630613', 'stderr': '', 'delta': '0:00:00.167398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms', ' 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:05.145512', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms\n 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms\n 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:04.899500', 'stderr': '', 'delta': '0:00:00.246012', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms', ' 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms', ' 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.595067', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.390 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.938 ms * *\n 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:56.437557', 'stderr': '', 'delta': '0:00:05.157510', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.390 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.938 ms * *', ' 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.992282', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:50.891009', 'stderr': '', 'delta': '0:00:00.101273', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:09.559098', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms\n 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *\n11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms\n14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms\n15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms\n16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms\n17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * dns.google (8.8.8.8)  71.765 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:51.894115', 'stderr': '', 'delta': '0:00:17.664983', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms', ' 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *', '11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms', '14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms', '15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms', '16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms', '17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * dns.google (8.8.8.8)  71.765 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.865573', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.297 ms * *\n 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:05.699120', 'stderr': '', 'delta': '0:00:05.166453', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.297 ms * *', ' 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.096748', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms\n 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms\n 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms\n12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms\n13  * * *\n14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms\n15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms\n16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms\n17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms\n18  * 142.250.56.125 (142.250.56.125)  57.256 ms *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  62.642 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:58.557954', 'stderr': '', 'delta': '0:00:33.538794', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms', ' 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms', ' 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms', '12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms', '13  * * *', '14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms', '15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms', '16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms', '17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms', '18  * 142.250.56.125 (142.250.56.125)  57.256 ms *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  62.642 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:20.194771', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms\n 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms\n 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms\n12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms\n13  * * *\n14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms\n15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms\n16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms\n17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  * dns.google (8.8.8.8)  58.098 ms *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:02.367537', 'stderr': '', 'delta': '0:00:17.827234', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms', ' 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms', ' 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms', '12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms', '13  * * *', '14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms', '15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms', '16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms', '17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  * dns.google (8.8.8.8)  58.098 ms *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:31.549055', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms\n 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms\n11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms\n12  * * *\n13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms\n14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms\n15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms\n16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms\n17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:08.705574', 'stderr': '', 'delta': '0:00:22.843481', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms', ' 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms', '11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms', '12  * * *', '13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms', '14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms', '15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms', '16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms', '17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.911963', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms\n 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms\n 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.679080', 'stderr': '', 'delta': '0:00:00.232883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms', ' 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms', ' 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.333705', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms\n 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:10.129179', 'stderr': '', 'delta': '0:00:00.204526', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms', ' 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.003986', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms\n 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms\n 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:20.751765', 'stderr': '', 'delta': '0:00:00.252221', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms', ' 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms', ' 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.326106', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms\n 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.156689', 'stderr': '', 'delta': '0:00:00.169417', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms', ' 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:33.680225', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms\n 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:33.492148', 'stderr': '', 'delta': '0:00:00.188077', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms', ' 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.999929', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:10.904827', 'stderr': '', 'delta': '0:00:00.095102', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.758829', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms\n 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:21.565598', 'stderr': '', 'delta': '0:00:00.193231', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms', ' 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.991750', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:32.871025', 'stderr': '', 'delta': '0:00:00.120725', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.418244', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.300 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.536 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:34.239759', 'stderr': '', 'delta': '0:00:05.178485', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.300 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.536 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:16.774332', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.328 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:11.559921', 'stderr': '', 'delta': '0:00:05.214411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.328 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:27.489632', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.386 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.664 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:22.327217', 'stderr': '', 'delta': '0:00:05.162415', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.386 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.664 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:38.664600', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.317 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:33.550717', 'stderr': '', 'delta': '0:00:05.113883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.317 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:17.597341', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms\n 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms\n 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:17.350816', 'stderr': '', 'delta': '0:00:00.246525', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms', ' 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms', ' 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.362887', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms\n 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms\n 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms\n 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:40.024809', 'stderr': '', 'delta': '0:00:00.338078', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms', ' 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms', ' 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms', ' 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:28.362078', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms\n 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms\n 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms\n 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:28.041002', 'stderr': '', 'delta': '0:00:00.321076', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms', ' 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms', ' 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms', ' 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.491805', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms\n 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms\n 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:39.235456', 'stderr': '', 'delta': '0:00:00.256349', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms', ' 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms', ' 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.303128', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:18.165153', 'stderr': '', 'delta': '0:00:00.137975', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.034733', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.938392', 'stderr': '', 'delta': '0:00:00.096341', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.022034', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:28.931194', 'stderr': '', 'delta': '0:00:00.090840', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.354126', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms\n 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms\n 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.088715', 'stderr': '', 'delta': '0:00:00.265411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms', ' 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms', ' 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.699115', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:41.601303', 'stderr': '', 'delta': '0:00:00.097812', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.969631', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:18.860114', 'stderr': '', 'delta': '0:00:00.109517', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.134107', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.314 ms * *\n 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms\n 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:40.948085', 'stderr': '', 'delta': '0:00:05.186022', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.314 ms * *', ' 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms', ' 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.721345', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:29.609710', 'stderr': '', 'delta': '0:00:00.111635', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:42.531026', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms\n 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms\n 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:42.283575', 'stderr': '', 'delta': '0:00:00.247451', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms', ' 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms', ' 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:19.735228', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms\n 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:19.530407', 'stderr': '', 'delta': '0:00:00.204821', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms', ' 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.798328', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:46.704208', 'stderr': '', 'delta': '0:00:00.094120', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:30.519888', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms\n 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms\n 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:30.269366', 'stderr': '', 'delta': '0:00:00.250522', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms', ' 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms', ' 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})

TASK [../roles/intermediateTestOfNetworkConnectivity : test | strore trace result] ***
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.715594', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms\n 2  * * *\n 3  * * *\n 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms\n 5  * * *\n 6  * * *\n 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms\n 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms\n 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms\n10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms\n11  * * *\n12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms\n13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms\n14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms\n15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms\n16  * 172.253.51.187 (172.253.51.187)  49.992 ms *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  dns.google (8.8.8.8)  65.551 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.714654', 'stderr': '', 'delta': '0:00:23.000940', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms', ' 2  * * *', ' 3  * * *', ' 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms', ' 5  * * *', ' 6  * * *', ' 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms', ' 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms', ' 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms', '10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms', '11  * * *', '12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms', '13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms', '14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms', '15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms', '16  * 172.253.51.187 (172.253.51.187)  49.992 ms *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  dns.google (8.8.8.8)  65.551 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.188820', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms\n 2  * * *\n 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms\n 4  * * *\n 5  * * *\n 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms\n 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms\n 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms\n 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms\n10  * * *\n11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms\n12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms\n13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms\n14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms\n15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *\n16  * * *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.679519', 'stderr': '', 'delta': '0:00:19.509301', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms', ' 2  * * *', ' 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms', ' 4  * * *', ' 5  * * *', ' 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms', ' 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms', ' 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms', ' 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms', '10  * * *', '11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms', '12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms', '13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms', '14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms', '15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *', '16  * * *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.781617', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms\n 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms\n 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms\n12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms\n13  * * *\n14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms\n15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms\n16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms\n17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms\n18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.716127', 'stderr': '', 'delta': '0:00:23.065490', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms', ' 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms', ' 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms', '12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms', '13  * * *', '14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms', '15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms', '16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms', '17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms', '18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.292177', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms\n 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms\n11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms\n12  * * *\n13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms\n14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms\n15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms\n16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.699399', 'stderr': '', 'delta': '0:00:27.592778', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms', ' 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms', '11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms', '12  * * *', '13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms', '14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms', '15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms', '16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.463805', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms\n 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms\n11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms\n14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms\n15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms\n16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms\n17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  64.830 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.772193', 'stderr': '', 'delta': '0:00:24.691612', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms', ' 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms', '11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms', '14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms', '15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms', '16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms', '17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  64.830 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.439494', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.318405', 'stderr': '', 'delta': '0:00:00.121089', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.344975', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms\n 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:48.145386', 'stderr': '', 'delta': '0:00:05.199589', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms', ' 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.853054', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:42.766411', 'stderr': '', 'delta': '0:00:00.086643', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.669218', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms\n 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms\n 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.375676', 'stderr': '', 'delta': '0:00:00.293542', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms', ' 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms', ' 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.098501', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms\n 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:50.926503', 'stderr': '', 'delta': '0:00:00.171998', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms', ' 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.147842', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.038789', 'stderr': '', 'delta': '0:00:00.109053', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.089207', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:53.994077', 'stderr': '', 'delta': '0:00:00.095130', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.444010', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms\n 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.241063', 'stderr': '', 'delta': '0:00:00.202947', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms', ' 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:43.566372', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:43.440441', 'stderr': '', 'delta': '0:00:00.125931', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:52.010000', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:51.910405', 'stderr': '', 'delta': '0:00:00.099595', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.286737', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.367 ms * *\n 2  192.168.0.33 (192.168.0.33)  2.995 ms * *\n 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:48.111492', 'stderr': '', 'delta': '0:00:05.175245', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.367 ms * *', ' 2  192.168.0.33 (192.168.0.33)  2.995 ms * *', ' 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.230515', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  * * *\n 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:44.134035', 'stderr': '', 'delta': '0:00:05.096480', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  * * *', ' 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.909389', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:47.808126', 'stderr': '', 'delta': '0:00:00.101263', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.770104', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:54.681551', 'stderr': '', 'delta': '0:00:00.088553', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.771173', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.327 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:52.572549', 'stderr': '', 'delta': '0:00:10.198624', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.327 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.048111', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:53.949070', 'stderr': '', 'delta': '0:00:00.099041', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:48.739323', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms\n 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:48.532562', 'stderr': '', 'delta': '0:00:00.206761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms', ' 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.069715', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms\n 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms\n 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:49.843490', 'stderr': '', 'delta': '0:00:00.226225', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms', ' 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms', ' 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:00.590285', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.342 ms * *\n 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms\n 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:55.414887', 'stderr': '', 'delta': '0:00:05.175398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.342 ms * *', ' 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms', ' 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.965468', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms\n 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms\n 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms\n 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:54.658707', 'stderr': '', 'delta': '0:00:00.306761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms', ' 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms', ' 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms', ' 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:03.480218', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:03.365980', 'stderr': '', 'delta': '0:00:00.114238', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.067606', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms\n 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms\n 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:50.808896', 'stderr': '', 'delta': '0:00:00.258710', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms', ' 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms', ' 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.470074', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms\n 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:49.316163', 'stderr': '', 'delta': '0:00:00.153911', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms', ' 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.430540', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms\n 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms\n 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:01.153780', 'stderr': '', 'delta': '0:00:00.276760', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms', ' 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms', ' 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:04.324858', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms\n 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms\n 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:04.075955', 'stderr': '', 'delta': '0:00:00.248903', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms', ' 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms', ' 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:55.867393', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms\n 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms\n 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms\n 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:55.572293', 'stderr': '', 'delta': '0:00:00.295100', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms', ' 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms', ' 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms', ' 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.283435', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms\n 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms\n 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:02.047439', 'stderr': '', 'delta': '0:00:00.235996', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms', ' 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms', ' 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.052522', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *\n 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms\n 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:51.893676', 'stderr': '', 'delta': '0:00:05.158846', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *', ' 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms', ' 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.188329', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms\n 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:50.035875', 'stderr': '', 'delta': '0:00:00.152454', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms', ' 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:05.145512', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms\n 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms\n 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:04.899500', 'stderr': '', 'delta': '0:00:00.246012', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms', ' 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms', ' 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.595067', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.390 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.938 ms * *\n 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:56.437557', 'stderr': '', 'delta': '0:00:05.157510', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.390 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.938 ms * *', ' 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.798011', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms\n 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:57.630613', 'stderr': '', 'delta': '0:00:00.167398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms', ' 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:07.968414', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.480 ms * *\n 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:02.860980', 'stderr': '', 'delta': '0:00:05.107434', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.480 ms * *', ' 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.992282', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:50.891009', 'stderr': '', 'delta': '0:00:00.101273', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.865573', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.297 ms * *\n 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:05.699120', 'stderr': '', 'delta': '0:00:05.166453', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.297 ms * *', ' 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:09.559098', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms\n 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *\n11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms\n14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms\n15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms\n16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms\n17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * dns.google (8.8.8.8)  71.765 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:51.894115', 'stderr': '', 'delta': '0:00:17.664983', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms', ' 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *', '11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms', '14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms', '15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms', '16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms', '17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * dns.google (8.8.8.8)  71.765 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:20.194771', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms\n 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms\n 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms\n12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms\n13  * * *\n14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms\n15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms\n16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms\n17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  * dns.google (8.8.8.8)  58.098 ms *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:02.367537', 'stderr': '', 'delta': '0:00:17.827234', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms', ' 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms', ' 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms', '12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms', '13  * * *', '14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms', '15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms', '16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms', '17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  * dns.google (8.8.8.8)  58.098 ms *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.096748', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms\n 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms\n 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms\n12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms\n13  * * *\n14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms\n15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms\n16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms\n17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms\n18  * 142.250.56.125 (142.250.56.125)  57.256 ms *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  62.642 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:58.557954', 'stderr': '', 'delta': '0:00:33.538794', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms', ' 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms', ' 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms', '12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms', '13  * * *', '14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms', '15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms', '16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms', '17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms', '18  * 142.250.56.125 (142.250.56.125)  57.256 ms *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  62.642 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:31.549055', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms\n 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms\n11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms\n12  * * *\n13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms\n14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms\n15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms\n16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms\n17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:08.705574', 'stderr': '', 'delta': '0:00:22.843481', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms', ' 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms', '11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms', '12  * * *', '13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms', '14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms', '15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms', '16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms', '17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.333705', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms\n 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:10.129179', 'stderr': '', 'delta': '0:00:00.204526', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms', ' 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.003986', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms\n 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms\n 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:20.751765', 'stderr': '', 'delta': '0:00:00.252221', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms', ' 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms', ' 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.911963', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms\n 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms\n 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.679080', 'stderr': '', 'delta': '0:00:00.232883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms', ' 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms', ' 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.326106', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms\n 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.156689', 'stderr': '', 'delta': '0:00:00.169417', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms', ' 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.999929', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:10.904827', 'stderr': '', 'delta': '0:00:00.095102', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.758829', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms\n 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:21.565598', 'stderr': '', 'delta': '0:00:00.193231', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms', ' 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:33.680225', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms\n 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:33.492148', 'stderr': '', 'delta': '0:00:00.188077', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms', ' 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.991750', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:32.871025', 'stderr': '', 'delta': '0:00:00.120725', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:27.489632', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.386 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.664 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:22.327217', 'stderr': '', 'delta': '0:00:05.162415', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.386 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.664 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:16.774332', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.328 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:11.559921', 'stderr': '', 'delta': '0:00:05.214411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.328 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.418244', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.300 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.536 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:34.239759', 'stderr': '', 'delta': '0:00:05.178485', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.300 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.536 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:38.664600', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.317 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:33.550717', 'stderr': '', 'delta': '0:00:05.113883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.317 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:17.597341', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms\n 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms\n 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:17.350816', 'stderr': '', 'delta': '0:00:00.246525', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms', ' 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms', ' 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:28.362078', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms\n 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms\n 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms\n 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:28.041002', 'stderr': '', 'delta': '0:00:00.321076', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms', ' 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms', ' 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms', ' 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.362887', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms\n 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms\n 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms\n 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:40.024809', 'stderr': '', 'delta': '0:00:00.338078', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms', ' 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms', ' 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms', ' 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.491805', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms\n 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms\n 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:39.235456', 'stderr': '', 'delta': '0:00:00.256349', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms', ' 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms', ' 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.303128', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:18.165153', 'stderr': '', 'delta': '0:00:00.137975', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.022034', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:28.931194', 'stderr': '', 'delta': '0:00:00.090840', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.034733', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.938392', 'stderr': '', 'delta': '0:00:00.096341', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.354126', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms\n 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms\n 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.088715', 'stderr': '', 'delta': '0:00:00.265411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms', ' 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms', ' 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.721345', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:29.609710', 'stderr': '', 'delta': '0:00:00.111635', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.969631', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:18.860114', 'stderr': '', 'delta': '0:00:00.109517', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.699115', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:41.601303', 'stderr': '', 'delta': '0:00:00.097812', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.134107', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.314 ms * *\n 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms\n 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:40.948085', 'stderr': '', 'delta': '0:00:05.186022', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.314 ms * *', ' 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms', ' 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:30.519888', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms\n 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms\n 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:30.269366', 'stderr': '', 'delta': '0:00:00.250522', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms', ' 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms', ' 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:19.735228', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms\n 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:19.530407', 'stderr': '', 'delta': '0:00:00.204821', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms', ' 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:42.531026', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms\n 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms\n 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:42.283575', 'stderr': '', 'delta': '0:00:00.247451', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms', ' 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms', ' 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.798328', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:46.704208', 'stderr': '', 'delta': '0:00:00.094120', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})

TASK [../roles/intermediateTestOfNetworkConnectivity : yum uninstall traceroute] ***
changed: [appRouter]
changed: [internetRouter]
changed: [webServer]
changed: [centralRouter]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
appServer                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centralRouter              : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbBackupServer             : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbServer                   : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
internetRouter             : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
monitoringServer           : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webServer                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>



<details><summary>см. iptables --table filter --list</summary>

```text
iptables --table filter --list
-------------------------------
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http flags:FIN,SYN,RST,ACK/SYN ctstate NEW
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https flags:FIN,SYN,RST,ACK/SYN ctstate NEW
ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:irdmi flags:FIN,SYN,RST,ACK/SYN ctstate NEW
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

```

</details>


<details><summary>см. iptables --table nat --list</summary>

```text
iptables --table nat --list
-------------------------------
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:192.168.0.2
DNAT       tcp  --  anywhere             anywhere             tcp dpt:https to:192.168.0.2
DNAT       tcp  --  anywhere             anywhere             tcp dpt:irdmi to:192.168.2.194

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  anywhere             anywhere            
SNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:http to:192.168.255.1
SNAT       tcp  --  anywhere             192.168.0.2          tcp dpt:https to:192.168.255.1
SNAT       tcp  --  anywhere             192.168.2.194        tcp dpt:irdmi to:192.168.255.1

```

</details>

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


<details><summary>см. intermediateTestOfNetworkConnectivity.yml</summary>

```text

PLAY [Playbook of test network connectivity] ***********************************

TASK [Gathering Facts] *********************************************************
ok: [webServer]
ok: [centralRouter]
ok: [appRouter]
ok: [appServer]
ok: [internetRouter]
ok: [dbRouter]
ok: [dbServer]
ok: [monitoringServer]
ok: [dbBackupServer]

TASK [../roles/intermediateTestOfNetworkConnectivity : yum install traceroute] ***
changed: [internetRouter]
changed: [centralRouter]
changed: [appServer]
changed: [webServer]
changed: [appRouter]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]
changed: [dbServer]

TASK [../roles/intermediateTestOfNetworkConnectivity : test | demostration] ****
ok: [internetRouter] => {
    "msg": "I am internetRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [centralRouter] => {
    "msg": "I am centralRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [webServer] => {
    "msg": "I am webServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [appRouter] => {
    "msg": "I am appRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [appServer] => {
    "msg": "I am appServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbRouter] => {
    "msg": "I am dbRouter, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbServer] => {
    "msg": "I am dbServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [dbBackupServer] => {
    "msg": "I am dbBackupServer, I get self name from `inventory_hostname` ansible-variable."
}
ok: [monitoringServer] => {
    "msg": "I am monitoringServer, I get self name from `inventory_hostname` ansible-variable."
}

TASK [../roles/intermediateTestOfNetworkConnectivity : test | traceroute foreign host] ***
changed: [internetRouter] => (item=8.8.8.8)
changed: [internetRouter] => (item=192.168.255.1)
changed: [internetRouter] => (item=192.168.255.2)
changed: [centralRouter] => (item=8.8.8.8)
changed: [appServer] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.255.1)
changed: [appServer] => (item=192.168.255.1)
changed: [centralRouter] => (item=192.168.255.2)
changed: [appServer] => (item=192.168.255.2)
changed: [webServer] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.0.2)
changed: [centralRouter] => (item=192.168.2.194)
changed: [internetRouter] => (item=192.168.0.2)
changed: [centralRouter] => (item=192.168.1.194)
changed: [internetRouter] => (item=192.168.2.194)
changed: [centralRouter] => (item=192.168.1.195)
changed: [appRouter] => (item=8.8.8.8)
changed: [centralRouter] => (item=192.168.0.66)
changed: [internetRouter] => (item=192.168.1.194)
changed: [appRouter] => (item=192.168.255.1)
changed: [appRouter] => (item=192.168.255.2)
changed: [appServer] => (item=192.168.0.2)
changed: [webServer] => (item=192.168.255.1)
changed: [appServer] => (item=192.168.2.194)
changed: [webServer] => (item=192.168.255.2)
changed: [webServer] => (item=192.168.0.2)
changed: [appServer] => (item=192.168.1.194)
changed: [appServer] => (item=192.168.1.195)
changed: [internetRouter] => (item=192.168.1.195)
changed: [internetRouter] => (item=192.168.0.66)
changed: [webServer] => (item=192.168.2.194)
changed: [webServer] => (item=192.168.1.194)
changed: [appServer] => (item=192.168.0.66)
changed: [webServer] => (item=192.168.1.195)
changed: [appRouter] => (item=192.168.0.2)
changed: [appRouter] => (item=192.168.2.194)
changed: [appRouter] => (item=192.168.1.194)
changed: [appRouter] => (item=192.168.1.195)
changed: [webServer] => (item=192.168.0.66)
changed: [dbRouter] => (item=8.8.8.8)
changed: [dbRouter] => (item=192.168.255.1)
changed: [appRouter] => (item=192.168.0.66)
changed: [dbRouter] => (item=192.168.255.2)
changed: [dbRouter] => (item=192.168.0.2)
changed: [dbRouter] => (item=192.168.2.194)
changed: [dbRouter] => (item=192.168.1.194)
changed: [dbRouter] => (item=192.168.1.195)
changed: [dbRouter] => (item=192.168.0.66)
changed: [dbBackupServer] => (item=8.8.8.8)
changed: [dbBackupServer] => (item=192.168.255.1)
changed: [dbBackupServer] => (item=192.168.255.2)
changed: [dbBackupServer] => (item=192.168.0.2)
changed: [dbBackupServer] => (item=192.168.2.194)
changed: [dbBackupServer] => (item=192.168.1.194)
changed: [dbBackupServer] => (item=192.168.1.195)
changed: [dbBackupServer] => (item=192.168.0.66)
changed: [monitoringServer] => (item=8.8.8.8)
changed: [dbServer] => (item=8.8.8.8)
changed: [monitoringServer] => (item=192.168.255.1)
changed: [dbServer] => (item=192.168.255.1)
changed: [monitoringServer] => (item=192.168.255.2)
changed: [dbServer] => (item=192.168.255.2)
changed: [monitoringServer] => (item=192.168.0.2)
changed: [dbServer] => (item=192.168.0.2)
changed: [monitoringServer] => (item=192.168.2.194)
changed: [monitoringServer] => (item=192.168.1.194)
changed: [dbServer] => (item=192.168.2.194)
changed: [dbServer] => (item=192.168.1.194)
changed: [dbServer] => (item=192.168.1.195)
changed: [dbServer] => (item=192.168.0.66)
changed: [monitoringServer] => (item=192.168.1.195)
changed: [monitoringServer] => (item=192.168.0.66)

TASK [../roles/intermediateTestOfNetworkConnectivity : test | delete preverios report] ***
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.463805', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms\n 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms\n11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms\n14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms\n15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms\n16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms\n17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  64.830 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.772193', 'stderr': '', 'delta': '0:00:24.691612', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms', ' 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms', '11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms', '14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms', '15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms', '16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms', '17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  64.830 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.715594', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms\n 2  * * *\n 3  * * *\n 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms\n 5  * * *\n 6  * * *\n 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms\n 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms\n 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms\n10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms\n11  * * *\n12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms\n13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms\n14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms\n15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms\n16  * 172.253.51.187 (172.253.51.187)  49.992 ms *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  dns.google (8.8.8.8)  65.551 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.714654', 'stderr': '', 'delta': '0:00:23.000940', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms', ' 2  * * *', ' 3  * * *', ' 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms', ' 5  * * *', ' 6  * * *', ' 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms', ' 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms', ' 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms', '10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms', '11  * * *', '12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms', '13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms', '14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms', '15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms', '16  * 172.253.51.187 (172.253.51.187)  49.992 ms *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  dns.google (8.8.8.8)  65.551 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.781617', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms\n 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms\n 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms\n12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms\n13  * * *\n14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms\n15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms\n16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms\n17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms\n18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.716127', 'stderr': '', 'delta': '0:00:23.065490', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms', ' 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms', ' 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms', '12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms', '13  * * *', '14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms', '15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms', '16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms', '17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms', '18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.188820', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms\n 2  * * *\n 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms\n 4  * * *\n 5  * * *\n 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms\n 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms\n 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms\n 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms\n10  * * *\n11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms\n12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms\n13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms\n14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms\n15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *\n16  * * *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.679519', 'stderr': '', 'delta': '0:00:19.509301', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms', ' 2  * * *', ' 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms', ' 4  * * *', ' 5  * * *', ' 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms', ' 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms', ' 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms', ' 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms', '10  * * *', '11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms', '12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms', '13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms', '14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms', '15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *', '16  * * *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.292177', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms\n 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms\n11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms\n12  * * *\n13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms\n14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms\n15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms\n16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.699399', 'stderr': '', 'delta': '0:00:27.592778', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms', ' 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms', '11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms', '12  * * *', '13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms', '14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms', '15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms', '16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.344975', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms\n 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:48.145386', 'stderr': '', 'delta': '0:00:05.199589', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms', ' 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.439494', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.318405', 'stderr': '', 'delta': '0:00:00.121089', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.853054', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:42.766411', 'stderr': '', 'delta': '0:00:00.086643', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.669218', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms\n 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms\n 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.375676', 'stderr': '', 'delta': '0:00:00.293542', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms', ' 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms', ' 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.098501', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms\n 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:50.926503', 'stderr': '', 'delta': '0:00:00.171998', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms', ' 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.089207', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:53.994077', 'stderr': '', 'delta': '0:00:00.095130', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:43.566372', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:43.440441', 'stderr': '', 'delta': '0:00:00.125931', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.147842', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.038789', 'stderr': '', 'delta': '0:00:00.109053', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.444010', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms\n 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.241063', 'stderr': '', 'delta': '0:00:00.202947', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms', ' 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:52.010000', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:51.910405', 'stderr': '', 'delta': '0:00:00.099595', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.770104', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:54.681551', 'stderr': '', 'delta': '0:00:00.088553', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.230515', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  * * *\n 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:44.134035', 'stderr': '', 'delta': '0:00:05.096480', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  * * *', ' 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.909389', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:47.808126', 'stderr': '', 'delta': '0:00:00.101263', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.286737', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.367 ms * *\n 2  192.168.0.33 (192.168.0.33)  2.995 ms * *\n 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:48.111492', 'stderr': '', 'delta': '0:00:05.175245', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.367 ms * *', ' 2  192.168.0.33 (192.168.0.33)  2.995 ms * *', ' 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.771173', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.327 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:52.572549', 'stderr': '', 'delta': '0:00:10.198624', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.327 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:00.590285', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.342 ms * *\n 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms\n 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:55.414887', 'stderr': '', 'delta': '0:00:05.175398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.342 ms * *', ' 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms', ' 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.069715', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms\n 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms\n 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:49.843490', 'stderr': '', 'delta': '0:00:00.226225', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms', ' 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms', ' 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:48.739323', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms\n 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:48.532562', 'stderr': '', 'delta': '0:00:00.206761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms', ' 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.048111', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:53.949070', 'stderr': '', 'delta': '0:00:00.099041', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.430540', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms\n 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms\n 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:01.153780', 'stderr': '', 'delta': '0:00:00.276760', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms', ' 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms', ' 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.067606', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms\n 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms\n 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:50.808896', 'stderr': '', 'delta': '0:00:00.258710', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms', ' 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms', ' 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:03.480218', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:03.365980', 'stderr': '', 'delta': '0:00:00.114238', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.470074', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms\n 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:49.316163', 'stderr': '', 'delta': '0:00:00.153911', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms', ' 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.965468', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms\n 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms\n 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms\n 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:54.658707', 'stderr': '', 'delta': '0:00:00.306761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms', ' 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms', ' 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms', ' 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.052522', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *\n 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms\n 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:51.893676', 'stderr': '', 'delta': '0:00:05.158846', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *', ' 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms', ' 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.283435', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms\n 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms\n 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:02.047439', 'stderr': '', 'delta': '0:00:00.235996', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms', ' 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms', ' 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:04.324858', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms\n 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms\n 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:04.075955', 'stderr': '', 'delta': '0:00:00.248903', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms', ' 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms', ' 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.188329', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms\n 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:50.035875', 'stderr': '', 'delta': '0:00:00.152454', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms', ' 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:55.867393', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms\n 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms\n 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms\n 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:55.572293', 'stderr': '', 'delta': '0:00:00.295100', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms', ' 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms', ' 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms', ' 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:07.968414', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.480 ms * *\n 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:02.860980', 'stderr': '', 'delta': '0:00:05.107434', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.480 ms * *', ' 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.798011', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms\n 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:57.630613', 'stderr': '', 'delta': '0:00:00.167398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms', ' 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:05.145512', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms\n 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms\n 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:04.899500', 'stderr': '', 'delta': '0:00:00.246012', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms', ' 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms', ' 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.595067', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.390 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.938 ms * *\n 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:56.437557', 'stderr': '', 'delta': '0:00:05.157510', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.390 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.938 ms * *', ' 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.992282', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:50.891009', 'stderr': '', 'delta': '0:00:00.101273', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:09.559098', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms\n 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *\n11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms\n14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms\n15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms\n16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms\n17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * dns.google (8.8.8.8)  71.765 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:51.894115', 'stderr': '', 'delta': '0:00:17.664983', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms', ' 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *', '11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms', '14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms', '15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms', '16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms', '17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * dns.google (8.8.8.8)  71.765 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.865573', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.297 ms * *\n 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:05.699120', 'stderr': '', 'delta': '0:00:05.166453', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.297 ms * *', ' 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.096748', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms\n 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms\n 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms\n12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms\n13  * * *\n14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms\n15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms\n16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms\n17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms\n18  * 142.250.56.125 (142.250.56.125)  57.256 ms *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  62.642 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:58.557954', 'stderr': '', 'delta': '0:00:33.538794', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms', ' 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms', ' 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms', '12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms', '13  * * *', '14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms', '15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms', '16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms', '17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms', '18  * 142.250.56.125 (142.250.56.125)  57.256 ms *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  62.642 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:20.194771', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms\n 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms\n 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms\n12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms\n13  * * *\n14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms\n15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms\n16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms\n17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  * dns.google (8.8.8.8)  58.098 ms *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:02.367537', 'stderr': '', 'delta': '0:00:17.827234', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms', ' 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms', ' 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms', '12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms', '13  * * *', '14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms', '15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms', '16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms', '17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  * dns.google (8.8.8.8)  58.098 ms *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:31.549055', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms\n 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms\n11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms\n12  * * *\n13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms\n14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms\n15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms\n16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms\n17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:08.705574', 'stderr': '', 'delta': '0:00:22.843481', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms', ' 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms', '11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms', '12  * * *', '13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms', '14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms', '15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms', '16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms', '17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.911963', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms\n 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms\n 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.679080', 'stderr': '', 'delta': '0:00:00.232883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms', ' 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms', ' 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.333705', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms\n 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:10.129179', 'stderr': '', 'delta': '0:00:00.204526', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms', ' 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.003986', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms\n 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms\n 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:20.751765', 'stderr': '', 'delta': '0:00:00.252221', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms', ' 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms', ' 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.326106', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms\n 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.156689', 'stderr': '', 'delta': '0:00:00.169417', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms', ' 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:33.680225', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms\n 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:33.492148', 'stderr': '', 'delta': '0:00:00.188077', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms', ' 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.999929', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:10.904827', 'stderr': '', 'delta': '0:00:00.095102', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.758829', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms\n 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:21.565598', 'stderr': '', 'delta': '0:00:00.193231', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms', ' 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.991750', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:32.871025', 'stderr': '', 'delta': '0:00:00.120725', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.418244', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.300 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.536 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:34.239759', 'stderr': '', 'delta': '0:00:05.178485', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.300 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.536 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:16.774332', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.328 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:11.559921', 'stderr': '', 'delta': '0:00:05.214411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.328 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:27.489632', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.386 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.664 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:22.327217', 'stderr': '', 'delta': '0:00:05.162415', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.386 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.664 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:38.664600', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.317 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:33.550717', 'stderr': '', 'delta': '0:00:05.113883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.317 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:17.597341', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms\n 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms\n 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:17.350816', 'stderr': '', 'delta': '0:00:00.246525', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms', ' 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms', ' 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.362887', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms\n 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms\n 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms\n 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:40.024809', 'stderr': '', 'delta': '0:00:00.338078', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms', ' 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms', ' 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms', ' 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:28.362078', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms\n 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms\n 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms\n 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:28.041002', 'stderr': '', 'delta': '0:00:00.321076', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms', ' 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms', ' 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms', ' 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.491805', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms\n 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms\n 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:39.235456', 'stderr': '', 'delta': '0:00:00.256349', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms', ' 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms', ' 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.303128', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:18.165153', 'stderr': '', 'delta': '0:00:00.137975', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.034733', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.938392', 'stderr': '', 'delta': '0:00:00.096341', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.022034', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:28.931194', 'stderr': '', 'delta': '0:00:00.090840', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.354126', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms\n 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms\n 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.088715', 'stderr': '', 'delta': '0:00:00.265411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms', ' 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms', ' 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.699115', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:41.601303', 'stderr': '', 'delta': '0:00:00.097812', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.969631', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:18.860114', 'stderr': '', 'delta': '0:00:00.109517', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.134107', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.314 ms * *\n 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms\n 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:40.948085', 'stderr': '', 'delta': '0:00:05.186022', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.314 ms * *', ' 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms', ' 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.721345', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:29.609710', 'stderr': '', 'delta': '0:00:00.111635', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
ok: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:42.531026', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms\n 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms\n 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:42.283575', 'stderr': '', 'delta': '0:00:00.247451', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms', ' 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms', ' 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:19.735228', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms\n 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:19.530407', 'stderr': '', 'delta': '0:00:00.204821', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms', ' 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.798328', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:46.704208', 'stderr': '', 'delta': '0:00:00.094120', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
ok: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:30.519888', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms\n 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms\n 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:30.269366', 'stderr': '', 'delta': '0:00:00.250522', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms', ' 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms', ' 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})

TASK [../roles/intermediateTestOfNetworkConnectivity : test | strore trace result] ***
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.715594', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms\n 2  * * *\n 3  * * *\n 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms\n 5  * * *\n 6  * * *\n 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms\n 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms\n 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms\n10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms\n11  * * *\n12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms\n13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms\n14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms\n15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms\n16  * 172.253.51.187 (172.253.51.187)  49.992 ms *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  dns.google (8.8.8.8)  65.551 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.714654', 'stderr': '', 'delta': '0:00:23.000940', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms', ' 2  * * *', ' 3  * * *', ' 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms', ' 5  * * *', ' 6  * * *', ' 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms', ' 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms', ' 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms', '10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms', '11  * * *', '12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms', '13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms', '14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms', '15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms', '16  * 172.253.51.187 (172.253.51.187)  49.992 ms *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  dns.google (8.8.8.8)  65.551 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.188820', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms\n 2  * * *\n 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms\n 4  * * *\n 5  * * *\n 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms\n 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms\n 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms\n 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms\n10  * * *\n11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms\n12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms\n13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms\n14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms\n15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *\n16  * * *\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.679519', 'stderr': '', 'delta': '0:00:19.509301', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms', ' 2  * * *', ' 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms', ' 4  * * *', ' 5  * * *', ' 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms', ' 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms', ' 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms', ' 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms', '10  * * *', '11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms', '12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms', '13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms', '14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms', '15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *', '16  * * *', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:45.781617', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms\n 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms\n 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms\n12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms\n13  * * *\n14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms\n15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms\n16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms\n17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms\n18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.716127', 'stderr': '', 'delta': '0:00:23.065490', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms', ' 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms', ' 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms', '12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms', '13  * * *', '14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms', '15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms', '16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms', '17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms', '18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.292177', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms\n 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms\n11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms\n12  * * *\n13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms\n14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms\n15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms\n16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms\n17  * * *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.699399', 'stderr': '', 'delta': '0:00:27.592778', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms', ' 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms', '11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms', '12  * * *', '13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms', '14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms', '15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms', '16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms', '17  * * *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.463805', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms\n 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms\n11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms\n14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms\n15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms\n16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms\n17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  dns.google (8.8.8.8)  64.830 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:22.772193', 'stderr': '', 'delta': '0:00:24.691612', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms', ' 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms', '11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms', '14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms', '15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms', '16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms', '17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  dns.google (8.8.8.8)  64.830 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.439494', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.318405', 'stderr': '', 'delta': '0:00:00.121089', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.344975', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms\n 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:48.145386', 'stderr': '', 'delta': '0:00:05.199589', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms', ' 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:42.853054', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:42.766411', 'stderr': '', 'delta': '0:00:00.086643', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:46.669218', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms\n 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms\n 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:46.375676', 'stderr': '', 'delta': '0:00:00.293542', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms', ' 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms', ' 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.098501', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms\n 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:22:50.926503', 'stderr': '', 'delta': '0:00:00.171998', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms', ' 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.147842', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.038789', 'stderr': '', 'delta': '0:00:00.109053', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.089207', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:53.994077', 'stderr': '', 'delta': '0:00:00.095130', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.444010', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms\n 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:47.241063', 'stderr': '', 'delta': '0:00:00.202947', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms', ' 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:43.566372', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:43.440441', 'stderr': '', 'delta': '0:00:00.125931', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:52.010000', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:22:51.910405', 'stderr': '', 'delta': '0:00:00.099595', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:53.286737', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.367 ms * *\n 2  192.168.0.33 (192.168.0.33)  2.995 ms * *\n 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:48.111492', 'stderr': '', 'delta': '0:00:05.175245', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.367 ms * *', ' 2  192.168.0.33 (192.168.0.33)  2.995 ms * *', ' 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.230515', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  * * *\n 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:44.134035', 'stderr': '', 'delta': '0:00:05.096480', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  * * *', ' 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:47.909389', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:47.808126', 'stderr': '', 'delta': '0:00:00.101263', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.770104', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:54.681551', 'stderr': '', 'delta': '0:00:00.088553', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.771173', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.327 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:22:52.572549', 'stderr': '', 'delta': '0:00:10.198624', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.327 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.048111', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:53.949070', 'stderr': '', 'delta': '0:00:00.099041', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:48.739323', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms\n 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:48.532562', 'stderr': '', 'delta': '0:00:00.206761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms', ' 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.069715', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms\n 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms\n 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:49.843490', 'stderr': '', 'delta': '0:00:00.226225', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms', ' 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms', ' 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:00.590285', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.342 ms * *\n 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms\n 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:22:55.414887', 'stderr': '', 'delta': '0:00:05.175398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.342 ms * *', ' 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms', ' 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:54.965468', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms\n 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms\n 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms\n 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:54.658707', 'stderr': '', 'delta': '0:00:00.306761', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms', ' 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms', ' 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms', ' 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:03.480218', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:03.365980', 'stderr': '', 'delta': '0:00:00.114238', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:51.067606', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms\n 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms\n 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:50.808896', 'stderr': '', 'delta': '0:00:00.258710', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms', ' 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms', ' 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:49.470074', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms\n 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:22:49.316163', 'stderr': '', 'delta': '0:00:00.153911', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms', ' 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.430540', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms\n 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms\n 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:01.153780', 'stderr': '', 'delta': '0:00:00.276760', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms', ' 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms', ' 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:04.324858', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms\n 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms\n 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:04.075955', 'stderr': '', 'delta': '0:00:00.248903', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms', ' 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms', ' 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:55.867393', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms\n 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms\n 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms\n 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:55.572293', 'stderr': '', 'delta': '0:00:00.295100', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.369 ms  0.199 ms  0.182 ms', ' 2  192.168.0.33 (192.168.0.33)  0.860 ms  0.733 ms  0.618 ms', ' 3  192.168.0.35 (192.168.0.35)  1.763 ms  1.661 ms  1.536 ms', ' 4  192.168.1.195 (192.168.1.195)  2.639 ms  2.475 ms  2.336 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:02.283435', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms\n 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms\n 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:02.047439', 'stderr': '', 'delta': '0:00:00.235996', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.564 ms  0.367 ms  0.224 ms', ' 2  192.168.0.35 (192.168.0.35)  1.025 ms  0.847 ms  0.955 ms', ' 3  192.168.1.195 (192.168.1.195)  2.555 ms  2.423 ms  2.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.052522', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *\n 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms\n 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:51.893676', 'stderr': '', 'delta': '0:00:05.158846', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.372 ms  0.212 ms *', ' 2  192.168.0.35 (192.168.0.35)  2.567 ms  2.434 ms  2.624 ms', ' 3  192.168.1.195 (192.168.1.195)  3.557 ms  4.191 ms  4.062 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.188329', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms\n 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:22:50.035875', 'stderr': '', 'delta': '0:00:00.152454', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.0.35 (192.168.0.35)  0.344 ms  0.132 ms  0.246 ms', ' 2  192.168.1.195 (192.168.1.195)  1.244 ms  1.158 ms  1.085 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:05.145512', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms\n 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms\n 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:04.899500', 'stderr': '', 'delta': '0:00:00.246012', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.347 ms  0.619 ms  0.409 ms', ' 2  192.168.0.35 (192.168.0.35)  0.496 ms  1.175 ms  1.752 ms', ' 3  192.168.1.195 (192.168.1.195)  1.733 ms  1.590 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [appServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:01.595067', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.2.193)  0.390 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.938 ms * *\n 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:56.437557', 'stderr': '', 'delta': '0:00:05.157510', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.2.193)  0.390 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.938 ms * *', ' 3  192.168.0.66 (192.168.0.66)  2.943 ms  2.810 ms  2.668 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [internetRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:57.798011', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms\n 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:57.630613', 'stderr': '', 'delta': '0:00:00.167398', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.361 ms  0.187 ms  0.202 ms', ' 2  192.168.0.66 (192.168.0.66)  0.498 ms  0.709 ms  2.029 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [webServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:07.968414', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.1)  0.480 ms * *\n 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:02.860980', 'stderr': '', 'delta': '0:00:05.107434', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.1)  0.480 ms * *', ' 2  192.168.0.66 (192.168.0.66)  2.254 ms  2.376 ms  2.237 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [centralRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:22:50.992282', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:22:50.891009', 'stderr': '', 'delta': '0:00:00.101273', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  192.168.0.66 (192.168.0.66)  0.333 ms  0.384 ms  0.209 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [appRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.865573', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.297 ms * *\n 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:05.699120', 'stderr': '', 'delta': '0:00:05.166453', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.297 ms * *', ' 2  192.168.0.66 (192.168.0.66)  0.761 ms  0.855 ms  0.719 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:09.559098', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms\n 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *\n11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms\n12  * * *\n13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms\n14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms\n15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms\n16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms\n17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * dns.google (8.8.8.8)  71.765 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:51.894115', 'stderr': '', 'delta': '0:00:17.664983', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms', ' 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *', '11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms', '12  * * *', '13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms', '14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms', '15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms', '16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms', '17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * dns.google (8.8.8.8)  71.765 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:20.194771', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms\n 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms\n 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms\n12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms\n13  * * *\n14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms\n15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms\n16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms\n17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  * dns.google (8.8.8.8)  58.098 ms *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:02.367537', 'stderr': '', 'delta': '0:00:17.827234', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms', ' 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms', ' 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms', '12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms', '13  * * *', '14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms', '15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms', '16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms', '17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  * dns.google (8.8.8.8)  58.098 ms *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.096748', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms\n 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms\n 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  * * *\n 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms\n10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms\n11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms\n12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms\n13  * * *\n14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms\n15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms\n16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms\n17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms\n18  * 142.250.56.125 (142.250.56.125)  57.256 ms *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  62.642 ms * *', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:22:58.557954', 'stderr': '', 'delta': '0:00:33.538794', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.280 ms  0.257 ms  0.241 ms', ' 2  192.168.0.33 (192.168.0.33)  1.670 ms  1.819 ms  4.633 ms', ' 3  192.168.255.1 (192.168.255.1)  5.629 ms  5.488 ms  5.378 ms', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  * * *', ' 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  42.540 ms  70.700 ms  70.579 ms', '10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  81.353 ms  81.187 ms  81.101 ms', '11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  80.972 ms  80.805 ms  80.678 ms', '12  74.125.49.108 (74.125.49.108)  87.635 ms  82.155 ms  88.821 ms', '13  * * *', '14  209.85.252.220 (209.85.252.220)  70.838 ms 74.125.244.129 (74.125.244.129)  70.694 ms  70.566 ms', '15  74.125.244.133 (74.125.244.133)  67.150 ms 74.125.244.180 (74.125.244.180)  70.289 ms  70.436 ms', '16  72.14.232.85 (72.14.232.85)  59.396 ms 216.239.48.163 (216.239.48.163)  57.929 ms 72.14.232.85 (72.14.232.85)  49.999 ms', '17  216.239.42.23 (216.239.42.23)  60.281 ms 172.253.51.187 (172.253.51.187)  58.212 ms 142.251.51.187 (142.251.51.187)  58.092 ms', '18  * 142.250.56.125 (142.250.56.125)  57.256 ms *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  62.642 ms * *'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:31.549055', 'stdout': 'traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms\n 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms\n 3  * * *\n 4  * * *\n 5  * * *\n 6  * * *\n 7  * * *\n 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms\n 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms\n10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms\n11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms\n12  * * *\n13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms\n14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms\n15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms\n16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms\n17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *\n18  * * *\n19  * * *\n20  * * *\n21  * * *\n22  * * *\n23  * * *\n24  * * *\n25  * * *\n26  * * *\n27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms', 'cmd': 'traceroute 8.8.8.8', 'rc': 0, 'start': '2021-11-30 13:23:08.705574', 'stderr': '', 'delta': '0:00:22.843481', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 8.8.8.8', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms', ' 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms', ' 3  * * *', ' 4  * * *', ' 5  * * *', ' 6  * * *', ' 7  * * *', ' 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms', ' 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms', '10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms', '11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms', '12  * * *', '13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms', '14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms', '15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms', '16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms', '17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *', '18  * * *', '19  * * *', '20  * * *', '21  * * *', '22  * * *', '23  * * *', '24  * * *', '25  * * *', '26  * * *', '27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms'], 'stderr_lines': [], 'failed': False, 'item': '8.8.8.8', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.333705', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms\n 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:10.129179', 'stderr': '', 'delta': '0:00:00.204526', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms', ' 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.003986', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms\n 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms\n 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:20.751765', 'stderr': '', 'delta': '0:00:00.252221', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms', ' 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms', ' 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.911963', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms\n 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms\n 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.679080', 'stderr': '', 'delta': '0:00:00.232883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.323 ms  0.230 ms  0.238 ms', ' 2  192.168.0.33 (192.168.0.33)  1.786 ms  2.665 ms  2.537 ms', ' 3  192.168.255.1 (192.168.255.1)  3.917 ms  3.830 ms  3.693 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.326106', 'stdout': 'traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms\n 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms', 'cmd': 'traceroute 192.168.255.1', 'rc': 0, 'start': '2021-11-30 13:23:32.156689', 'stderr': '', 'delta': '0:00:00.169417', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.1', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms', ' 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.1', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:10.999929', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:10.904827', 'stderr': '', 'delta': '0:00:00.095102', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:21.758829', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms\n 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:21.565598', 'stderr': '', 'delta': '0:00:00.193231', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms', ' 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:33.680225', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms\n 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:33.492148', 'stderr': '', 'delta': '0:00:00.188077', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  2.424 ms  2.185 ms  2.026 ms', ' 2  192.168.255.2 (192.168.255.2)  3.227 ms  3.113 ms  2.993 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:32.991750', 'stdout': 'traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets\n 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms', 'cmd': 'traceroute 192.168.255.2', 'rc': 0, 'start': '2021-11-30 13:23:32.871025', 'stderr': '', 'delta': '0:00:00.120725', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.255.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets', ' 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.255.2', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:27.489632', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.386 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.664 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:22.327217', 'stderr': '', 'delta': '0:00:05.162415', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.386 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.664 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:16.774332', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.328 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:11.559921', 'stderr': '', 'delta': '0:00:05.214411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.328 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.418244', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.300 ms * *\n 2  192.168.0.33 (192.168.0.33)  0.536 ms * *\n 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:34.239759', 'stderr': '', 'delta': '0:00:05.178485', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.300 ms * *', ' 2  192.168.0.33 (192.168.0.33)  0.536 ms * *', ' 3  192.168.0.2 (192.168.0.2)  1.625 ms  3.194 ms  3.071 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:38.664600', 'stdout': 'traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.317 ms * *\n 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms', 'cmd': 'traceroute 192.168.0.2', 'rc': 0, 'start': '2021-11-30 13:23:33.550717', 'stderr': '', 'delta': '0:00:05.113883', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.2', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.317 ms * *', ' 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.2', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:17.597341', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms\n 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms\n 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:17.350816', 'stderr': '', 'delta': '0:00:00.246525', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms', ' 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms', ' 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:28.362078', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms\n 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms\n 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms\n 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:28.041002', 'stderr': '', 'delta': '0:00:00.321076', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms', ' 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms', ' 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms', ' 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.362887', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms\n 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms\n 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms\n 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:40.024809', 'stderr': '', 'delta': '0:00:00.338078', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.888 ms  0.636 ms  0.517 ms', ' 2  192.168.0.33 (192.168.0.33)  1.861 ms  4.485 ms  4.364 ms', ' 3  192.168.0.34 (192.168.0.34)  4.081 ms  3.933 ms  3.812 ms', ' 4  192.168.2.194 (192.168.2.194)  3.677 ms  3.546 ms  3.406 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:39.491805', 'stdout': 'traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms\n 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms\n 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms', 'cmd': 'traceroute 192.168.2.194', 'rc': 0, 'start': '2021-11-30 13:23:39.235456', 'stderr': '', 'delta': '0:00:00.256349', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.2.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms', ' 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms', ' 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.2.194', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.303128', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:18.165153', 'stderr': '', 'delta': '0:00:00.137975', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.022034', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:28.931194', 'stderr': '', 'delta': '0:00:00.090840', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.034733', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.938392', 'stderr': '', 'delta': '0:00:00.096341', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  dbServer (192.168.1.194)  0.044 ms  0.014 ms  0.012 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:40.354126', 'stdout': 'traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms\n 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms\n 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms', 'cmd': 'traceroute 192.168.1.194', 'rc': 0, 'start': '2021-11-30 13:23:40.088715', 'stderr': '', 'delta': '0:00:00.265411', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.194', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms', ' 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms', ' 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.194', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:29.721345', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:29.609710', 'stderr': '', 'delta': '0:00:00.111635', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  dbBackupServer (192.168.1.195)  0.046 ms  0.014 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:18.969631', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:18.860114', 'stderr': '', 'delta': '0:00:00.109517', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.355 ms  0.128 ms  0.294 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:41.699115', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:41.601303', 'stderr': '', 'delta': '0:00:00.097812', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  192.168.1.195 (192.168.1.195)  0.318 ms  0.265 ms  0.238 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.134107', 'stdout': 'traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.65)  0.314 ms * *\n 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms\n 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms', 'cmd': 'traceroute 192.168.1.195', 'rc': 0, 'start': '2021-11-30 13:23:40.948085', 'stderr': '', 'delta': '0:00:05.186022', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.1.195', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.1.195 (192.168.1.195), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.65)  0.314 ms * *', ' 2  192.168.0.35 (192.168.0.35)  0.587 ms  0.468 ms  0.969 ms', ' 3  192.168.1.195 (192.168.1.195)  1.458 ms  1.324 ms  2.102 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.1.195', 'ansible_loop_var': 'item'})
changed: [dbBackupServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:30.519888', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms\n 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms\n 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:30.269366', 'stderr': '', 'delta': '0:00:00.250522', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.374 ms  0.172 ms  0.203 ms', ' 2  192.168.0.33 (192.168.0.33)  0.614 ms  0.723 ms  3.824 ms', ' 3  192.168.0.66 (192.168.0.66)  3.692 ms  3.555 ms  3.352 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbRouter -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:19.735228', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms\n 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:19.530407', 'stderr': '', 'delta': '0:00:00.204821', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.0.33)  0.330 ms  0.129 ms  0.653 ms', ' 2  192.168.0.66 (192.168.0.66)  1.172 ms  1.039 ms  0.902 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [dbServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:42.531026', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms\n 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms\n 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:42.283575', 'stderr': '', 'delta': '0:00:00.247451', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  gateway (192.168.1.193)  0.363 ms  0.143 ms  0.213 ms', ' 2  192.168.0.33 (192.168.0.33)  0.490 ms  0.368 ms  2.248 ms', ' 3  192.168.0.66 (192.168.0.66)  2.115 ms  1.921 ms  1.780 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})
changed: [monitoringServer -> localhost] => (item={'changed': True, 'end': '2021-11-30 13:23:46.798328', 'stdout': 'traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets\n 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms', 'cmd': 'traceroute 192.168.0.66', 'rc': 0, 'start': '2021-11-30 13:23:46.704208', 'stderr': '', 'delta': '0:00:00.094120', 'invocation': {'module_args': {'creates': None, 'executable': None, '_uses_shell': True, 'strip_empty_ends': True, '_raw_params': 'traceroute 192.168.0.66', 'removes': None, 'argv': None, 'warn': False, 'chdir': None, 'stdin_add_newline': True, 'stdin': None}}, 'stdout_lines': ['traceroute to 192.168.0.66 (192.168.0.66), 30 hops max, 60 byte packets', ' 1  monitoringServer (192.168.0.66)  0.043 ms  0.013 ms  0.013 ms'], 'stderr_lines': [], 'failed': False, 'item': '192.168.0.66', 'ansible_loop_var': 'item'})

TASK [../roles/intermediateTestOfNetworkConnectivity : yum uninstall traceroute] ***
changed: [appRouter]
changed: [internetRouter]
changed: [webServer]
changed: [centralRouter]
changed: [appServer]
changed: [dbServer]
changed: [dbRouter]
changed: [dbBackupServer]
changed: [monitoringServer]

PLAY RECAP *********************************************************************
appRouter                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
appServer                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centralRouter              : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbBackupServer             : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbServer                   : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
internetRouter             : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
monitoringServer           : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
webServer                  : ok=7    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

##### internetRouter


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (10.0.2.2)  0.846 ms  1.270 ms  3.885 ms
 2  * * *
 3  10.17.135.122 (10.17.135.122)  61.589 ms  78.968 ms  77.972 ms
 4  * * *
 5  * * *
 6  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  75.824 ms  73.400 ms  75.702 ms
 7  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  74.510 ms  74.214 ms  71.377 ms
 8  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  70.774 ms  73.259 ms  72.788 ms
 9  74.125.49.108 (74.125.49.108)  74.631 ms  72.279 ms  73.615 ms
10  * * *
11  209.85.240.254 (209.85.240.254)  74.626 ms 216.239.59.142 (216.239.59.142)  73.905 ms 209.85.252.220 (209.85.252.220)  73.838 ms
12  74.125.244.181 (74.125.244.181)  72.343 ms 74.125.244.180 (74.125.244.180)  65.776 ms 74.125.244.181 (74.125.244.181)  64.711 ms
13  72.14.232.84 (72.14.232.84)  65.100 ms 142.251.61.219 (142.251.61.219)  69.476 ms 72.14.232.84 (72.14.232.84)  43.303 ms
14  142.251.61.219 (142.251.61.219)  43.868 ms 172.253.51.237 (172.253.51.237)  65.864 ms 216.239.48.163 (216.239.48.163)  63.178 ms
15  142.250.56.13 (142.250.56.13)  60.389 ms 142.250.56.219 (142.250.56.219)  62.873 ms *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * dns.google (8.8.8.8)  69.053 ms  68.354 ms

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  internetRouter (192.168.255.1)  0.075 ms  0.028 ms  0.053 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.383 ms  0.356 ms  0.343 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  * * *
 2  192.168.0.2 (192.168.0.2)  0.465 ms  0.358 ms  0.817 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.295 ms  0.310 ms  0.384 ms
 2  192.168.0.34 (192.168.0.34)  1.823 ms  1.712 ms  1.530 ms
 3  192.168.2.194 (192.168.2.194)  1.386 ms  2.035 ms  1.918 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.430 ms  0.396 ms  0.360 ms
 2  192.168.0.35 (192.168.0.35)  0.597 ms  0.703 ms  0.886 ms
 3  192.168.1.194 (192.168.1.194)  1.450 ms  2.078 ms  2.051 ms

```

</details>

#### centralRouter


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.255.1)  0.392 ms  0.195 ms  0.238 ms
 2  * * *
 3  * * *
 4  10.17.135.122 (10.17.135.122)  60.310 ms  60.090 ms  64.150 ms
 5  * * *
 6  * * *
 7  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  60.798 ms  60.507 ms  63.585 ms
 8  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  63.427 ms  76.993 ms  73.723 ms
 9  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  72.717 ms  72.516 ms  66.200 ms
10  74.125.49.108 (74.125.49.108)  77.275 ms  65.236 ms  74.288 ms
11  * * *
12  74.125.244.129 (74.125.244.129)  63.077 ms  68.258 ms 74.125.37.218 (74.125.37.218)  64.491 ms
13  74.125.244.181 (74.125.244.181)  69.181 ms 74.125.244.180 (74.125.244.180)  39.110 ms 74.125.244.133 (74.125.244.133)  60.876 ms
14  142.251.61.221 (142.251.61.221)  40.450 ms 72.14.232.85 (72.14.232.85)  57.539 ms 142.251.61.221 (142.251.61.221)  42.801 ms
15  216.239.49.3 (216.239.49.3)  83.575 ms 216.239.56.101 (216.239.56.101)  57.183 ms 142.251.51.187 (142.251.51.187)  62.360 ms
16  * 172.253.51.187 (172.253.51.187)  49.992 ms *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  dns.google (8.8.8.8)  65.551 ms * *

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.255.1)  1.794 ms  1.455 ms  1.283 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  centralRouter (192.168.255.2)  0.043 ms  0.013 ms  0.015 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  192.168.0.2 (192.168.0.2)  0.347 ms  0.243 ms  0.282 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  192.168.0.34 (192.168.0.34)  0.357 ms  0.162 ms  0.179 ms
 2  192.168.2.194 (192.168.2.194)  0.585 ms  1.606 ms  1.474 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  192.168.0.35 (192.168.0.35)  0.379 ms  0.265 ms  1.253 ms
 2  192.168.1.194 (192.168.1.194)  1.137 ms  1.011 ms  1.396 ms

```

</details>

#### webServer


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.0.1)  0.323 ms  0.201 ms  0.261 ms
 2  192.168.255.1 (192.168.255.1)  0.459 ms  0.487 ms  0.520 ms
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.309 ms  58.104 ms  79.856 ms
 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  79.500 ms  79.052 ms  78.733 ms
10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  78.249 ms  90.247 ms  78.556 ms
11  74.125.49.108 (74.125.49.108)  78.283 ms  77.990 ms  77.663 ms
12  * * *
13  74.125.244.129 (74.125.244.129)  78.922 ms 216.239.59.142 (216.239.59.142)  57.595 ms 74.125.244.129 (74.125.244.129)  78.188 ms
14  74.125.244.132 (74.125.244.132)  77.727 ms 74.125.244.180 (74.125.244.180)  77.342 ms 74.125.244.133 (74.125.244.133)  77.108 ms
15  72.14.232.85 (72.14.232.85)  76.012 ms 142.251.61.221 (142.251.61.221)  80.211 ms 72.14.232.84 (72.14.232.84)  68.589 ms
16  209.85.251.41 (209.85.251.41)  55.860 ms 142.251.51.187 (142.251.51.187)  62.453 ms 172.253.51.185 (172.253.51.185)  54.628 ms
17  74.125.253.147 (74.125.253.147)  61.854 ms * 172.253.70.47 (172.253.70.47)  71.531 ms
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  dns.google (8.8.8.8)  64.830 ms * *

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.0.1)  2.001 ms  1.891 ms  1.768 ms
 2  192.168.255.1 (192.168.255.1)  2.714 ms  2.776 ms  2.671 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.315 ms  0.249 ms  0.209 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  webServer (192.168.0.2)  0.036 ms  0.020 ms  0.012 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.1)  0.342 ms * *
 2  192.168.0.34 (192.168.0.34)  0.898 ms  1.517 ms  1.335 ms
 3  192.168.2.194 (192.168.2.194)  1.889 ms  2.648 ms  2.515 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.1)  0.292 ms  0.282 ms  0.231 ms
 2  192.168.0.35 (192.168.0.35)  0.799 ms  0.623 ms  1.711 ms
 3  192.168.1.194 (192.168.1.194)  2.099 ms  1.975 ms  2.118 ms

```

</details>

#### appRouter


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.499 ms  0.476 ms  0.487 ms
 2  192.168.255.1 (192.168.255.1)  4.766 ms  4.486 ms  4.287 ms
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  68.714 ms  75.108 ms  66.036 ms
 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  51.551 ms  50.550 ms  60.381 ms
10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  38.006 ms  40.039 ms  59.766 ms
11  74.125.49.108 (74.125.49.108)  38.147 ms  60.810 ms  42.128 ms
12  * * *
13  216.239.59.142 (216.239.59.142)  65.895 ms 209.85.252.220 (209.85.252.220)  65.598 ms 216.239.59.142 (216.239.59.142)  65.239 ms
14  74.125.244.181 (74.125.244.181)  47.096 ms  58.102 ms 74.125.244.132 (74.125.244.132)  60.957 ms
15  72.14.232.84 (72.14.232.84)  61.709 ms 72.14.232.85 (72.14.232.85)  61.460 ms 72.14.232.84 (72.14.232.84)  61.204 ms
16  142.250.210.103 (142.250.210.103)  60.863 ms 216.239.48.163 (216.239.48.163)  60.626 ms 142.251.61.219 (142.251.61.219)  61.306 ms
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  dns.google (8.8.8.8)  84.180 ms *  83.795 ms

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.282 ms  0.266 ms  0.164 ms
 2  192.168.255.1 (192.168.255.1)  0.619 ms  0.525 ms  0.384 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.330 ms  0.337 ms  0.515 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.327 ms * *
 2  192.168.0.2 (192.168.0.2)  0.708 ms  1.023 ms  0.864 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  192.168.2.194 (192.168.2.194)  0.827 ms  0.589 ms  0.364 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.370 ms  0.247 ms  0.379 ms
 2  192.168.0.35 (192.168.0.35)  2.553 ms  2.410 ms  2.233 ms
 3  192.168.1.194 (192.168.1.194)  2.087 ms  1.899 ms  1.764 ms

```

</details>

#### appServer


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.2.193)  13.350 ms  7.245 ms  9.946 ms
 2  192.168.0.33 (192.168.0.33)  9.814 ms  9.686 ms  9.538 ms
 3  192.168.255.1 (192.168.255.1)  9.346 ms  9.135 ms  8.918 ms
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  62.942 ms  69.267 ms  48.989 ms
10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  72.294 ms  77.266 ms  48.831 ms
11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  63.054 ms  47.519 ms  49.644 ms
12  74.125.49.108 (74.125.49.108)  48.029 ms *  97.208 ms
13  * * *
14  74.125.244.129 (74.125.244.129)  95.939 ms 216.239.59.142 (216.239.59.142)  94.903 ms  95.522 ms
15  74.125.244.133 (74.125.244.133)  89.132 ms 74.125.244.181 (74.125.244.181)  104.722 ms 74.125.244.180 (74.125.244.180)  88.238 ms
16  142.251.51.187 (142.251.51.187)  94.646 ms 142.251.61.219 (142.251.61.219)  94.607 ms 72.14.232.85 (72.14.232.85)  87.863 ms
17  142.251.51.187 (142.251.51.187)  94.176 ms 142.250.56.131 (142.250.56.131)  94.285 ms 216.239.62.9 (216.239.62.9)  93.434 ms
18  216.239.62.15 (216.239.62.15)  52.402 ms 172.253.51.243 (172.253.51.243)  52.048 ms 216.239.58.53 (216.239.58.53)  62.021 ms
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  dns.google (8.8.8.8)  70.423 ms *  68.875 ms

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.2.193)  0.299 ms  0.231 ms  0.176 ms
 2  192.168.0.33 (192.168.0.33)  2.558 ms  2.415 ms  2.308 ms
 3  192.168.255.1 (192.168.255.1)  2.849 ms  2.761 ms  2.651 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  gateway (192.168.2.193)  0.340 ms  0.432 ms  0.543 ms
 2  192.168.255.2 (192.168.255.2)  1.941 ms  3.264 ms  2.973 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  gateway (192.168.2.193)  0.367 ms * *
 2  192.168.0.33 (192.168.0.33)  2.995 ms * *
 3  192.168.0.2 (192.168.0.2)  3.325 ms  3.200 ms  3.085 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  appServer (192.168.2.194)  0.044 ms  0.019 ms  0.017 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  gateway (192.168.2.193)  2.485 ms  2.277 ms  2.132 ms
 2  192.168.0.33 (192.168.0.33)  4.322 ms  4.192 ms  4.038 ms
 3  192.168.0.35 (192.168.0.35)  6.739 ms  6.639 ms  6.514 ms
 4  192.168.1.194 (192.168.1.194)  7.552 ms  7.435 ms  7.326 ms

```

</details>

#### dbRouter


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.923 ms  0.660 ms  0.509 ms
 2  192.168.255.1 (192.168.255.1)  1.362 ms  1.169 ms  1.694 ms
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  54.127 ms  48.582 ms  60.305 ms
 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.463 ms  59.005 ms  58.825 ms
10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  59.581 ms  91.171 ms *
11  74.125.49.108 (74.125.49.108)  44.594 ms  68.697 ms  68.556 ms
12  * * *
13  74.125.244.129 (74.125.244.129)  68.002 ms 209.85.245.238 (209.85.245.238)  67.866 ms 74.125.37.218 (74.125.37.218)  67.742 ms
14  74.125.244.132 (74.125.244.132)  67.589 ms  66.551 ms 74.125.244.133 (74.125.244.133)  67.335 ms
15  142.251.61.221 (142.251.61.221)  69.347 ms 142.251.51.187 (142.251.51.187)  66.999 ms 142.251.61.221 (142.251.61.221)  69.031 ms
16  172.253.51.223 (172.253.51.223)  68.878 ms 142.251.61.221 (142.251.61.221)  90.586 ms 216.239.48.163 (216.239.48.163)  90.141 ms
17  * 216.239.62.107 (216.239.62.107)  89.783 ms 142.250.209.161 (142.250.209.161)  89.614 ms
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * dns.google (8.8.8.8)  71.765 ms

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.344 ms  0.130 ms  0.502 ms
 2  192.168.255.1 (192.168.255.1)  0.918 ms  0.785 ms  0.653 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.817 ms  0.589 ms  0.464 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.328 ms * *
 2  192.168.0.2 (192.168.0.2)  0.602 ms  0.740 ms  0.600 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.33)  0.367 ms  0.283 ms  0.332 ms
 2  192.168.0.34 (192.168.0.34)  0.446 ms  0.830 ms  1.541 ms
 3  192.168.2.194 (192.168.2.194)  1.920 ms  1.702 ms  1.699 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  192.168.1.194 (192.168.1.194)  0.525 ms  0.287 ms  0.242 ms

```

</details>

#### dbBackupServer


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.1.193)  0.427 ms  0.157 ms  0.196 ms
 2  192.168.0.33 (192.168.0.33)  1.060 ms  0.928 ms  0.806 ms
 3  192.168.255.1 (192.168.255.1)  4.157 ms  4.049 ms  3.926 ms
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  78.385 ms  49.388 ms  38.404 ms
10  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  60.185 ms  61.763 ms  64.803 ms
11  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  51.031 ms  51.345 ms  50.806 ms
12  74.125.49.108 (74.125.49.108)  50.360 ms  134.885 ms  54.252 ms
13  * * *
14  209.85.240.254 (209.85.240.254)  85.734 ms 74.125.244.129 (74.125.244.129)  92.192 ms 216.239.59.142 (216.239.59.142)  86.881 ms
15  74.125.244.180 (74.125.244.180)  90.707 ms  83.506 ms 74.125.244.132 (74.125.244.132)  91.180 ms
16  142.251.61.219 (142.251.61.219)  91.964 ms 72.14.232.84 (72.14.232.84)  89.859 ms 142.251.51.187 (142.251.51.187)  92.951 ms
17  142.250.56.127 (142.250.56.127)  92.517 ms 172.253.51.249 (172.253.51.249)  91.601 ms 172.253.51.189 (172.253.51.189)  92.430 ms
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * dns.google (8.8.8.8)  58.098 ms *

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.1.193)  0.493 ms  0.452 ms  0.299 ms
 2  192.168.0.33 (192.168.0.33)  0.987 ms  0.995 ms  1.764 ms
 3  192.168.255.1 (192.168.255.1)  1.638 ms  1.501 ms  1.358 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  gateway (192.168.1.193)  0.471 ms  0.160 ms  0.201 ms
 2  192.168.255.2 (192.168.255.2)  0.596 ms  0.610 ms  0.473 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  gateway (192.168.1.193)  0.386 ms * *
 2  192.168.0.33 (192.168.0.33)  0.664 ms * *
 3  192.168.0.2 (192.168.0.2)  1.205 ms  3.563 ms  3.393 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  gateway (192.168.1.193)  0.303 ms  0.113 ms  0.172 ms
 2  192.168.0.33 (192.168.0.33)  0.790 ms  0.682 ms  1.448 ms
 3  192.168.0.34 (192.168.0.34)  1.313 ms  1.363 ms  2.947 ms
 4  192.168.2.194 (192.168.2.194)  2.802 ms  2.918 ms  2.885 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  192.168.1.194 (192.168.1.194)  0.517 ms  0.263 ms  0.443 ms

```

</details>

#### monitoringServer


<details><summary>см. traceroute 8.8.8.8</summary>

```text
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  gateway (192.168.0.65)  0.385 ms  0.211 ms  0.089 ms
 2  192.168.255.1 (192.168.255.1)  0.501 ms  0.369 ms  0.970 ms
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  oct-cr03-be79.100.spb.mts-internet.net (212.188.18.189)  53.120 ms  70.464 ms  58.497 ms
 9  oct-cr01-be2.78.spb.mts-internet.net (212.188.1.101)  59.869 ms  56.063 ms  41.877 ms
10  bor-cr02-ae5.78.spb.mts-internet.net (212.188.28.14)  50.005 ms  50.424 ms  63.017 ms
11  74.125.49.108 (74.125.49.108)  55.903 ms  40.878 ms  57.392 ms
12  * * *
13  74.125.37.218 (74.125.37.218)  71.696 ms 74.125.244.129 (74.125.244.129)  71.841 ms  50.715 ms
14  74.125.244.180 (74.125.244.180)  50.352 ms  70.165 ms 74.125.244.133 (74.125.244.133)  76.934 ms
15  72.14.232.84 (72.14.232.84)  76.376 ms 216.239.48.163 (216.239.48.163)  76.020 ms 142.251.61.221 (142.251.61.221)  75.790 ms
16  216.239.48.163 (216.239.48.163)  75.559 ms 216.239.49.113 (216.239.49.113)  75.353 ms 142.251.61.219 (142.251.61.219)  75.109 ms
17  172.253.79.169 (172.253.79.169)  74.869 ms 216.239.56.113 (216.239.56.113)  74.638 ms *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  dns.google (8.8.8.8)  76.347 ms *  54.197 ms

```

</details>


<details><summary>см. traceroute 192.168.255.1</summary>

```text
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  gateway (192.168.0.65)  0.331 ms  0.192 ms  1.833 ms
 2  192.168.255.1 (192.168.255.1)  1.811 ms  1.683 ms  1.455 ms

```

</details>


<details><summary>см. traceroute 192.168.255.2</summary>

```text
traceroute to 192.168.255.2 (192.168.255.2), 30 hops max, 60 byte packets
 1  192.168.255.2 (192.168.255.2)  0.389 ms  0.195 ms  0.194 ms

```

</details>


<details><summary>см. traceroute 192.168.0.2</summary>

```text
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  gateway (192.168.0.65)  0.317 ms * *
 2  192.168.0.2 (192.168.0.2)  0.784 ms  0.627 ms  1.076 ms

```

</details>


<details><summary>см. traceroute 192.168.2.194</summary>

```text
traceroute to 192.168.2.194 (192.168.2.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.65)  0.345 ms  0.137 ms  0.207 ms
 2  192.168.0.34 (192.168.0.34)  1.978 ms  2.526 ms  2.407 ms
 3  192.168.2.194 (192.168.2.194)  2.272 ms  2.722 ms  2.746 ms

```

</details>


<details><summary>см. traceroute 192.168.1.194</summary>

```text
traceroute to 192.168.1.194 (192.168.1.194), 30 hops max, 60 byte packets
 1  gateway (192.168.0.65)  0.325 ms  0.322 ms  0.301 ms
 2  192.168.0.35 (192.168.0.35)  1.775 ms  1.570 ms  1.363 ms
 3  192.168.1.194 (192.168.1.194)  1.563 ms  1.385 ms  1.650 ms

```

</details>

#### Настройка бд

```shell
ansible-playbook playbooks/postgresql.yml --tags deploy > ../reports/playbooks/postgresql.txt
```


<details><summary>см. playbooks/postgresql.yml</summary>

```text

PLAY [Playbook of db configure] ************************************************

TASK [Gathering Facts] *********************************************************
ok: [dbServer]

TASK [../roles/postgresql : Install EPEL Repo package from standart repo] ******
changed: [dbServer]

TASK [../roles/postgresql : Install PostgreSQL repo] ***************************
changed: [dbServer]

TASK [../roles/postgresql : Yum install] ***************************************
changed: [dbServer] => (item=postgresql13-server)
changed: [dbServer] => (item=python3)
changed: [dbServer] => (item=python-pip)
changed: [dbServer] => (item=python-psycopg2)

TASK [../roles/postgresql : pip & pip3 install pexpect | ISSUE] ****************
changed: [dbServer] => (item=pip)
changed: [dbServer] => (item=pip3)

TASK [../roles/postgresql : Init PostgreSQL] ***********************************
changed: [dbServer]

TASK [../roles/postgresql : Collect-pg.conf-files] *****************************
changed: [dbServer] => (item=pg_hba.conf)
changed: [dbServer] => (item=postgresql.conf)

TASK [../roles/postgresql : Force restart PostgreSQL] **************************
changed: [dbServer]

TASK [../roles/postgresql : Create project database] ***************************
changed: [dbServer]

TASK [../roles/postgresql : Create postgres user for project] ******************
changed: [dbServer]

TASK [../roles/postgresql : Ensure we have access from the new user] ***********
ok: [dbServer]

TASK [../roles/postgresql : Check local PostgreSQL access] *********************
changed: [dbServer]

TASK [../roles/postgresql : Print result of check local PostgreSQL access] *****
ok: [dbServer] => {
    "msg": {
        "changed": true,
        "cmd": "psql -c 'SELECT version()' -U admin -h 127.0.0.1 project -W",
        "delta": "0:00:00.250540",
        "end": "2021-11-30 13:30:06.952218",
        "failed": false,
        "rc": 0,
        "start": "2021-11-30 13:30:06.701678",
        "stdout": "Пароль: \r\n                                                 version                        \r\n                         \r\n--------------------------------------------------------------------------------\r\n-------------------------\r\n PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R\r\ned Hat 4.8.5-44), 64-bit\r\n(1 строка)",
        "stdout_lines": [
            "Пароль: ",
            "                                                 version                        ",
            "                         ",
            "--------------------------------------------------------------------------------",
            "-------------------------",
            " PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R",
            "ed Hat 4.8.5-44), 64-bit",
            "(1 строка)"
        ]
    }
}

TASK [../roles/postgresql : Store result of check local PostgreSQL access] *****
changed: [dbServer -> localhost]

RUNNING HANDLER [../roles/postgresql : restart-postgresql] *********************
changed: [dbServer]

PLAY RECAP *********************************************************************
dbServer                   : ok=15   changed=12   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

```shell
ansible-playbook playbooks/intermediateTestOfDbAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfDbAvailabilityFromExternalHost.txt
```


<details><summary>см. playbooks/intermediateTestOfDbAvailabilityFromExternalHost.yml</summary>

```text

PLAY [Playbook of intermediate test of external db connection] *****************

TASK [Gathering Facts] *********************************************************
ok: [appServer]
ok: [dbRouter]

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Install EPEL Repo package from standart repo] ***
changed: [dbRouter]
changed: [appServer]

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Install PostgreSQL repo] ***
changed: [dbRouter]
changed: [appServer]

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Yum install] ***
changed: [appServer] => (item=postgresql13)
changed: [dbRouter] => (item=postgresql13)
changed: [appServer] => (item=python3)
changed: [dbRouter] => (item=python3)
changed: [appServer] => (item=python-pip)
changed: [dbRouter] => (item=python-pip)

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : pip & pip3 install pexpect] ***
changed: [appServer] => (item=pip)
changed: [dbRouter] => (item=pip)
changed: [appServer] => (item=pip3)
changed: [dbRouter] => (item=pip3)

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Check remote db access] ***
changed: [appServer]
changed: [dbRouter]

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Print result of check remote db access] ***
ok: [dbRouter] => {
    "msg": {
        "changed": true,
        "cmd": "psql -c 'SELECT version()' -U admin -h 192.168.1.194 project -W",
        "delta": "0:00:00.247329",
        "end": "2021-11-30 13:34:34.875805",
        "failed": false,
        "rc": 0,
        "start": "2021-11-30 13:34:34.628476",
        "stdout": "Пароль: \r\n                                                 version                        \r\n                         \r\n--------------------------------------------------------------------------------\r\n-------------------------\r\n PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R\r\ned Hat 4.8.5-44), 64-bit\r\n(1 строка)",
        "stdout_lines": [
            "Пароль: ",
            "                                                 version                        ",
            "                         ",
            "--------------------------------------------------------------------------------",
            "-------------------------",
            " PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R",
            "ed Hat 4.8.5-44), 64-bit",
            "(1 строка)"
        ]
    }
}
ok: [appServer] => {
    "msg": {
        "changed": true,
        "cmd": "psql -c 'SELECT version()' -U admin -h 192.168.1.194 project -W",
        "delta": "0:00:00.245639",
        "end": "2021-11-30 13:34:34.894914",
        "failed": false,
        "rc": 0,
        "start": "2021-11-30 13:34:34.649275",
        "stdout": "Пароль: \r\n                                                 version                        \r\n                         \r\n--------------------------------------------------------------------------------\r\n-------------------------\r\n PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R\r\ned Hat 4.8.5-44), 64-bit\r\n(1 строка)",
        "stdout_lines": [
            "Пароль: ",
            "                                                 version                        ",
            "                         ",
            "--------------------------------------------------------------------------------",
            "-------------------------",
            " PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R",
            "ed Hat 4.8.5-44), 64-bit",
            "(1 строка)"
        ]
    }
}

TASK [../roles/intermediateTestOfDbAvailabilityFromExternalHost : Store result of check remote db access] ***
changed: [dbRouter -> localhost]
changed: [appServer -> localhost]

PLAY RECAP *********************************************************************
appServer                  : ok=8    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
dbRouter                   : ok=8    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>



<details><summary>см. local access check</summary>

```text
psql -c 'SELECT version()' -U admin -h 127.0.0.1 project -W
-------------------------------
Пароль: 
                                                 version                        
                         
--------------------------------------------------------------------------------
-------------------------
 PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R
ed Hat 4.8.5-44), 64-bit
(1 строка)

```

</details>

<details><summary>см. remote access test from dbRouter</summary>

```text
psql -c 'SELECT version()' -U admin -h 192.168.1.194 project -W
-------------------------------
Пароль: 
                                                 version                        
                         
--------------------------------------------------------------------------------
-------------------------
 PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R
ed Hat 4.8.5-44), 64-bit
(1 строка)

```

</details>

<details><summary>см. remote access test from appServer</summary>

```text
psql -c 'SELECT version()' -U admin -h 192.168.1.194 project -W
-------------------------------
Пароль: 
                                                 version                        
                         
--------------------------------------------------------------------------------
-------------------------
 PostgreSQL 13.5 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (R
ed Hat 4.8.5-44), 64-bit
(1 строка)

```

</details>


#### Настройка nginx

```shell
ansible-playbook playbooks/nginx.yml --tags deploy_django > ../reports/playbooks/nginx.txt
ansible-playbook playbooks/intermediateTestOfWebAvailabilityFromExternalHost.yml> ./reports/playbooks/intermediateTestOfWebAvailabilityFromExternalHost.txt
```


<details><summary>см. playbooks/nginx.yml</summary>

```text

PLAY [Playbook of nginx configure] *********************************************

TASK [Gathering Facts] *********************************************************
ok: [webServer]

PLAY RECAP *********************************************************************
webServer                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>


<details><summary>см. playbooks/intermediateTestOfWebAvailabilityFromExternalHost.yml</summary>

```text

PLAY [Playbook of intermediate test web server availability] *******************

TASK [Gathering Facts] *********************************************************
ok: [centralRouter]

TASK [../roles/intermediateTestOfWebAvailabilityFromExternalHost : Check web response] ***
fatal: [centralRouter]: FAILED! => {"changed": false, "content": "", "elapsed": 0, "msg": "Status code was -1 and not [200]: Request failed: <urlopen error [Errno 111] В соединении отказано>", "redirected": false, "status": -1, "url": "http://192.168.0.2"}

PLAY RECAP *********************************************************************
centralRouter              : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   


```

</details>

#### Настройка Django

```shell
ansible-playbook playbooks/django.yml --tags deploy > ../reports/playbooks/django.txt
ansible-playbook playbooks/intermediateTestOfAppAvailabilityFromExternalHost.yml > ../reports/playbooks/intermediateTestOfAppAvailabilityFromExternalHost.txt
```


<details><summary>см. playbooks/django.yml</summary>

```text

PLAY [Playbook of django config] ***********************************************

TASK [Gathering Facts] *********************************************************
ok: [appServer]

TASK [../roles/django : Install EPEL Repo package from standart repo] **********
ok: [appServer]

TASK [../roles/django : Install PostgreSQL repo] *******************************
ok: [appServer]

TASK [../roles/django : Yum install] *******************************************
ok: [appServer] => (item=postgresql13)
ok: [appServer] => (item=python-pip)
ok: [appServer] => (item=python3)
changed: [appServer] => (item=python3-psycopg2)
changed: [appServer] => (item=python3-gunicorn)
changed: [appServer] => (item=django)

TASK [../roles/django : Pip3 install packages] *********************************
changed: [appServer] => (item=django)

TASK [../roles/django : Django application deploy] *****************************
changed: [appServer] => (item=/home/vagrant/django_app/)

TASK [../roles/django : Gunicorn service deploy] *******************************
changed: [appServer] => (item=/etc/systemd/system/)

TASK [../roles/django : Gunicorn verify] ***************************************
changed: [appServer]

TASK [../roles/django : Print verify-gunicorn] *********************************
ok: [appServer] => {
    "msg": {
        "changed": true,
        "cmd": "systemd-analyze verify gunicorn.service",
        "delta": "0:00:00.237593",
        "end": "2021-11-30 13:40:43.577131",
        "failed": false,
        "rc": 0,
        "start": "2021-11-30 13:40:43.339538",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "",
        "stdout_lines": []
    }
}

TASK [../roles/django : Force restart gunicorn] ********************************
changed: [appServer]

TASK [../roles/django : django manage.py chmod] ********************************
changed: [appServer]

TASK [../roles/django : django makemigrations] *********************************
ok: [appServer]

TASK [../roles/django : django migrate] ****************************************
changed: [appServer]

TASK [../roles/django : createsuperuser admin] *********************************
ok: [appServer]

RUNNING HANDLER [../roles/django : systemctl-restart-gunicorn] *****************
changed: [appServer]

PLAY RECAP *********************************************************************
appServer                  : ok=15   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

<details><summary>см. playbooks/intermediateTestOfAppAvailabilityFromExternalHost.yml</summary>

```text

PLAY [Playbook of intermediate test of app access] *****************************

TASK [Gathering Facts] *********************************************************
ok: [appRouter]

TASK [../roles/intermediateTestOfAppAvailabilityFromExternalHost : Check gunicorn/django response] ***
fatal: [appRouter]: FAILED! => {"changed": false, "connection": "close", "content": "\n<!doctype html>\n<html lang=\"en\">\n<head>\n  <title>Not Found</title>\n</head>\n<body>\n  <h1>Not Found</h1><p>The requested resource was not found on this server.</p>\n</body>\n</html>\n", "content_length": "179", "content_type": "text/html", "date": "Tue, 30 Nov 2021 13:41:03 GMT", "elapsed": 0, "msg": "Status code was 404 and not [200]: HTTP Error 404: Not Found", "redirected": false, "referrer_policy": "same-origin", "server": "gunicorn/20.0.4", "status": 404, "url": "http://192.168.2.194:8000", "x_content_type_options": "nosniff", "x_frame_options": "DENY"}

PLAY RECAP *********************************************************************
appRouter                  : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   


```

</details>

#### Настройка логгирования

```shell
ansible-playbook playbooks/rsyslog_server.yml --tags deploy > ../reports/playbooks/rsyslog_server.txt
ansible-playbook playbooks/rsyslog_clients.yml --tags clients > ../reports/playbooks/rsyslog_clients.txt
```


<details><summary>см. playbooks/rsyslog_server.yml</summary>

```text

PLAY [Playbook of rsyslog server configure] ************************************

TASK [Gathering Facts] *********************************************************
ok: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure UDP | Enable ModLoad imudp] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure UDP | Enable UDPServerRun 514] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | open UDP ports] **********************
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure TCP | Enable ModLoad imtcp] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure TCP | Enable InputTCPServerRun 514] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | open TCP ports] **********************
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Change log template formatting (make timestamp better)] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | clear /etc/rsyslog.d/ and /var/log/rsyslog/] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog program] *******************************
changed: [monitoringServer] => (item=nginx)
changed: [monitoringServer] => (item=django_project)
changed: [monitoringServer] => (item=postgresql)

RUNNING HANDLER [../roles/rsyslog_server : restart rsyslog] ********************
changed: [monitoringServer]

PLAY RECAP *********************************************************************
monitoringServer           : ok=11   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

<details><summary>см. playbooks/rsyslog_server.yml</summary>

```text

PLAY [Playbook of rsyslog server configure] ************************************

TASK [Gathering Facts] *********************************************************
ok: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure UDP | Enable ModLoad imudp] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure UDP | Enable UDPServerRun 514] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | open UDP ports] **********************
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure TCP | Enable ModLoad imtcp] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Configure TCP | Enable InputTCPServerRun 514] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | open TCP ports] **********************
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | Change log template formatting (make timestamp better)] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog | clear /etc/rsyslog.d/ and /var/log/rsyslog/] ***
changed: [monitoringServer]

TASK [../roles/rsyslog_server : Rsyslog program] *******************************
changed: [monitoringServer] => (item=nginx)
changed: [monitoringServer] => (item=django_project)
changed: [monitoringServer] => (item=postgresql)

RUNNING HANDLER [../roles/rsyslog_server : restart rsyslog] ********************
changed: [monitoringServer]

PLAY RECAP *********************************************************************
monitoringServer           : ok=11   changed=10   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

</details>

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

