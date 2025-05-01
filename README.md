# Развернуть отказоустойчивый кластер MySQL

## Задание

1. Разворачиваем отказоустойчивый кластер **MySQL** любым способом.
2. Создаем внутри кластера вашу базу данных для проекта.

## Реализация

Задание сделано так, чтобы его можно было запустить как в **Vagrant**, так и в **Yandex Cloud**. После запуска происходит развёртывание следующих виртуальных машин:

- **mysql-iscsi-01** - сервер **iSCSI target**;
- **mysql-iscsi-02** - сервер **iSCSI target**;
- **mysql-backend-01** - **GLPI**, клиент файловой системы **GFS2**;
- **mysql-backend-02** - **GLPI**, клиент файловой системы **GFS2**;
- **mysql-backend-03** - **GLPI**, клиент файловой системы **GFS2**;

В независимости от того, как созданы виртуальные машины, для их настройки запускается **Ansible Playbook** [provision.yml](provision.yml) который последовательно запускает следующие роли:

- **wait_connection** - ожидает доступность виртуальных машин.
- **apt_sources** - настраивает репозитории для пакетного менеджера **apt** (используется [mirror.yandex.ru](https://mirror.yandex.ru)).
- **bach_completion** - устанавливает пакет **bash-completion**.
- **chrony** - устанавливает **chrony** для синхронизации времени между узлами.
- **hosts** - прописывает адреса всех узлов в `/etc/hosts`.
- **gen_keys** - генерит `/etc/corosync/authkey` для кластера **corosync**.
- **disk_facts** - собирает информацию о дисках и их сигнатурах (с помощью утилит `lsblk` и `wipefs`).
- **disk_label** - разбивает диски и устанавливает на них **GPT Partition Label** для их дальнейшей идентификации.
- **target** - настраивает сервер **iSCSI Target**.
- **linux_modules** - устанавливает модули ядра (в **Yandex Cloud** стоит **linux-virtual**, который не содержит модулей ядра для работы с **GFS2**).
- **iscsi** - настраивает **iSCSI Initiator**.
- **mpath** - настраивает **multipathd**, в частности прописывает **reservation_key** в `/etc/multipath.conf` для последующего использования в агенте **fence_mpath** для настройки **fencing**'а.
- **corosync** - настраивает кластер **corosync** в несколько колец.
- **dlm** - устанавливает распределённый менеджер блокировок **dlm**.
- **mdadm** - устанавилвает **mdadm** и создаёт **RAID1** массив `/dev/md/cluster-md` (используется технология [MD Cluster](https://docs.kernel.org/driver-api/md/md-cluster.html)).
- **lvm_facts** - с помощью утилит **vgs** и **lvs** собирает информацию о группах и томах **lvm**;
- **lvm** - устанавливает **lvm2**, **lvm2-lockd**, создаёт группы томов, сами логические тома и активирует их.
- **gfs2** - устанавливает **gfs2-utils**.
- **filesystem** - форматирует общий диск в файловую систему **GFS2**.
- **directory** - создаёт пустую директорию `/var/lib/glpi`.
- **pacemaker** - устанавливает и настраивается **pacemaker**, который в свою очередь монтирует файловую систему `/dev/cluster-vg/cluster-lv` в `/var/lib/glpi`.
- **angie** - устанавливает и настраивает **angie**;
- **glpi** - устанавиливает и настраивает **glpi**;
- **mariadb** - устанавиливает и настраивает **mariadb**;
- **mariadb_databases** - создаёт базы данных в **mariadb**;
- **mariadb_users** - создаёт пользователей, для подключения к **mariadb**;
- **php_fpm** - устанавиливает и настраивает **php-fpm**;
- **system_groups** - создаёт группы пользователей в системе (в частности для **glpi**);
- **system_users** - создаёт пользователей в системе (в частности для **glpi**);
- **tls_ca** - создаёт сертификаты для корневых центров сертификации;
- **tls_certs** - создаёт сертификаты для узлов;
- **tls_copy** - копирует серитификаты на узел;
- **keepalived** - устанавливает и настраивает **keepalived** при разворачивании в **vagrant**.

Данные роли настраиваются с помощью переменных, определённых в следующих файлах:

- [group_vars/all/angie.yml](group_vars/all/angie.yml) - общие настройки **angie**;
- [group_vars/all/ansible.yml](group_vars/all/ansible.yml) - общие переменные **ansible** для всех узлов;
- [group_vars/all/certs.yml](group_vars/all/certs.yml) - настройки генерации сертификатов для СУБД и **angie**;
- [group_vars/all/glpi.yml](group_vars/all/glpi.yml) - общие настройки **GLPI**;
- [group_vars/all/hosts.yml](group_vars/all/hosts.yml) - настройки для роли **hosts** (список узлов, которые нужно добавить в `/etc/hosts`);
- [group_vars/all/iscsi.yml](group_vars/all/iscsi.yml) - общие настройки для **iSCSI Target** и **iSCSI Initiator**;
- [group_vars/backend/angie.yml](group_vars/backend/angie.yml) - настройки **angie** для узлов **backend**;
- [group_vars/backend/certs.yml](group_vars/backend/certs.yml) - настройки генерации сертификатов для **backend**;
- [group_vars/backend/corosync.yml](group_vars/backend/corosync.yml) - настройки **corosync**;
- [group_vars/backend/gfs2.yml](group_vars/backend/gfs2.yml) - настройки **MD Cluster**, **LVM**, **GFS2**;
- [group_vars/backend/glpi.yml](group_vars/backend/glpi.yml) - настройки **GLPI** для **backend**;
- [group_vars/backend/iscsi.yml](group_vars/backend/iscsi.yml) - настройки **iSCSI Initiator**;
- [group_vars/backend/keepalived.yml](group_vars/backend/keepalived.yml) - настройки **keepalived** для узлов **backend**;
- [group_vars/backend/mariadb.yml](group_vars/backend/mariadb.yml) - настройки **mariadb**;
- [group_vars/backend/php.yml](group_vars/backend/php.yml) - настройки **PHP** для **backend**;
- [group_vars/backend/users.yml](group_vars/backend/users.yml) - настройки создания пользователей и групп на узлах **backend**;
- [group_vars/iscsi/iscsi.yml](group_vars/iscsi/iscsi.yml) - настройки **iSCSI Target**;
- [host_vars/mysql-backend-01/gfs2.yml](host_vars/mysql-backend-01/gfs2.yml) - настройки создания **MD Cluster**, **LVM**, **GFS2**;
- [host_vars/mysql-backend-01/pacemaker.yml](host_vars/mysql-backend-01/pacemaker.yml) - настройки **pacemaker**;
- [host_vars/mysql-backend-01/mariadb.yml](host_vars/mysql-backend-01/mariadb.yml) - настройки **mariadb**;
- [host_vars/mysql-backend-01/keepalived.yml](host_vars/mysql-backend-01/keepalived.yml) - настройки **keepalived** для **mysql-backend-01**;
- [host_vars/mysql-backend-02/keepalived.yml](host_vars/mysql-backend-02/keepalived.yml) - настройки **keepalived** для **mysql-backend-02**;
- [host_vars/mysql-backend-03/keepalived.yml](host_vars/mysql-backend-03/keepalived.yml) - настройки **keepalived** для **mysql-backend-03**.

## Запуск

### Запуск в Yandex Cloud

1. Необходимо установить и настроить утилиту **yc** по инструкции [Начало работы с интерфейсом командной строки](https://yandex.cloud/ru/docs/cli/quickstart).
2. Необходимо установить **Terraform** по инструкции [Начало работы с Terraform](https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart).
3. Необходимо установить **Ansible**.
4. Необходимо перейти в папку проекта и запустить скрипт [up.sh](up.sh).

### Запуск в Vagrant (VirtualBox)

Необходимо скачать **VagrantBox** для **bento/ubuntu-24.04** версии **202502.21.0** и добавить его в **Vagrant** под именем **bento/ubuntu-24.04/202502.21.0**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/bento/boxes/ubuntu-24.04/versions/202502.21.0/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "bento/ubuntu-24.04/202502.21.0"
rm vagrant.box
```

После этого нужно сделать **vagrant up** в папке проекта.

## Проверка

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.4.5**
- **VirtualBox 7.1.8_SUSE r168469**
- **Ansible 2.18.5**
- **Python 3.13.3**
- **Jinja2 3.1.6**

После запуска **GLPI** должен открываться по **IP** балансировщика. Для **Yandex Cloud** адрес можно узнать в выводе **terraform** в поле **load_balancer** (смотри [outputs.tf](outputs.tf)). Для **vagrant** это (можно использовать любой адрес):

- [https://192.168.56.51](https://192.168.56.51) - узел **mysql-backend-01**.
- [https://192.168.56.52](https://192.168.56.52) - узел **mysql-backend-02**.
- [https://192.168.56.53](https://192.168.56.53) - узел **mysql-backend-03**.

Однако **keepalived** настроен таким образом, что при недоступности одного из узлов, его адрес переезжает на один из доступных.

Проверим работу кластера **mariadb** на одном из узлов, для начала проверим конфигурацию:

```text
❯ vagrant ssh mysql-backend-01 -c 'cat /etc/mysql/my.cnf'
[client-server]
socket = "/run/mysqld/mysqld.sock"

[mariadbd]
pid_file = "/run/mysqld/mysqld.pid"
basedir = "/usr"
bind-address = "127.0.0.1"
expire_logs_days = 10
server_id = 0
gtid_domain_id = "998211615"
gtid_strict_mode = 1
skip_name_resolve = 1
wait_timeout = 60
log_bin = 1
log_basename = "mariadb"
log_slave_updates = 1
binlog_expire_logs_seconds = 604800
max_binlog_size = "1G"
ssl_ca = "/etc/mysql/mariadb_ca.crt"
ssl_cert = "/etc/mysql/mariadb.crt"
ssl_key = "/etc/mysql/mariadb.key"
wsrep_on = 1
wsrep_provider = "/usr/lib/libgalera_smm.so"
wsrep_cluster_name = "GLPI"
wsrep_cluster_address = "gcomm://192.168.56.21,192.168.56.22,192.168.56.23"
wsrep_provider_options = "gmcast.listen_addr=tcp://192.168.56.21;socket.ssl_ca=/etc/mysql/mariadb_ca.crt;socket.ssl_cert=/etc/mysql/mariadb.crt;socket.ssl_key=/etc/mysql/mariadb.key"
wsrep_node_address = "192.168.56.21"
wsrep_node_name = "mysql-backend-01"
wsrep_slave_threads = 4
wsrep_sst_method = "mariabackup"
wsrep_sst_auth = "mysql:"
wsrep_gtid_mode = 1
wsrep_gtid_domain_id = 998211614
binlog_format = "ROW"
default_storage_engine = "InnoDB"
innodb_doublewrite = 1
innodb_flush_log_at_trx_commit = 0
innodb_autoinc_lock_mode = 2
```

Проверим, что служба запущена на всех узлах:

```text
❯ vagrant ssh mysql-backend-01 -c 'systemctl status mariadb.service'
● mariadb.service - MariaDB 11.7.2 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/mariadb.service.d
             └─migrated-from-my.cnf-settings.conf
     Active: active (running) since Thu 2025-05-01 14:34:53 UTC; 15min ago
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
   Main PID: 5754 (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 21 (limit: 15003)
     Memory: 285.9M (peak: 286.2M)
        CPU: 6.634s
     CGroup: /system.slice/mariadb.service
             └─5754 /usr/sbin/mariadbd --wsrep_start_position=4825d4e1-2699-11f0-8e08-87e21ae24a04:15,998211614-1-13

May 01 14:34:58 mysql-backend-01 mariadbd[5754]: =================================================
May 01 14:34:58 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:58 2 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
May 01 14:34:58 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:58 2 [Note] WSREP: Lowest cert index boundary for CC from group: 11
May 01 14:34:58 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:58 2 [Note] WSREP: Min available from gcache for CC from group: 1
May 01 14:34:59 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:59 0 [Note] WSREP: Member 2.0 (mysql-backend-03) requested state transfer >
May 01 14:34:59 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:59 0 [Note] WSREP: 1.0 (mysql-backend-02): State transfer to 2.0 (mysql-ba>
May 01 14:34:59 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:59 0 [Note] WSREP: Member 1.0 (mysql-backend-02) synced with group.
May 01 14:34:59 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:59 0 [Note] WSREP: 2.0 (mysql-backend-03): State transfer from 1.0 (mysql->
May 01 14:34:59 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:34:59 0 [Note] WSREP: Member 2.0 (mysql-backend-03) synced with group.
May 01 14:35:00 mysql-backend-01 mariadbd[5754]: 2025-05-01 14:35:00 0 [Note] WSREP: (717b2058-9d50, 'ssl://192.168.56.21:4567') turning mes>

❯ vagrant ssh mysql-backend-02 -c 'systemctl status mariadb.service'
● mariadb.service - MariaDB 11.7.2 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/mariadb.service.d
             └─migrated-from-my.cnf-settings.conf
     Active: active (running) since Thu 2025-05-01 14:34:56 UTC; 15min ago
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
   Main PID: 6468 (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 20 (limit: 15003)
     Memory: 287.8M (peak: 288.1M)
        CPU: 6.136s
     CGroup: /system.slice/mariadb.service
             └─6468 /usr/sbin/mariadbd --wsrep_start_position=4825d4e1-2699-11f0-8e08-87e21ae24a04:17,998211614-1-13

May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 8 [Note] WSREP: Synchronized with group, ready for connections
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 8 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
May 01 14:34:59 mysql-backend-02 mariadbd[6811]: WSREP_SST: [INFO] mariabackup IST completed on donor (20250501 14:34:59.344)
May 01 14:34:59 mysql-backend-02 mariadbd[6811]: WSREP_SST: [INFO] Cleaning up temporary directories (20250501 14:34:59.347)
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 0 [Note] WSREP: Donor monitor thread ended with total time 0 sec
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 0 [Note] WSREP: Cleaning up SST user.
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 0 [Note] WSREP: async IST sender served
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 0 [Note] WSREP: 2.0 (mysql-backend-03): State transfer from 1.0 (mysql->
May 01 14:34:59 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:34:59 0 [Note] WSREP: Member 2.0 (mysql-backend-03) synced with group.
May 01 14:35:01 mysql-backend-02 mariadbd[6468]: 2025-05-01 14:35:01 0 [Note] WSREP: (73148389-8e06, 'ssl://192.168.56.22:4567') turning mes>

❯ vagrant ssh mysql-backend-03 -c 'systemctl status mariadb.service'
● mariadb.service - MariaDB 11.7.2 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/mariadb.service.d
             └─migrated-from-my.cnf-settings.conf
     Active: active (running) since Thu 2025-05-01 14:34:59 UTC; 15min ago
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
   Main PID: 6743 (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 20 (limit: 15003)
     Memory: 288.2M (peak: 288.5M)
        CPU: 5.924s
     CGroup: /system.slice/mariadb.service
             └─6743 /usr/sbin/mariadbd --wsrep_start_position=4825d4e1-2699-11f0-8e08-87e21ae24a04:19,998211614-1-13

May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 0 [Note] WSREP: Member 2.0 (mysql-backend-03) synced with group.
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 0 [Note] WSREP: Processing event queue:... 100.0% (1/1 events) complete.
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 0 [Note] WSREP: Shifting JOINED -> SYNCED (TO: 21)
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 2 [Note] WSREP: Server mysql-backend-03 synced with group
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 2 [Note] WSREP: Server status change joined -> synced
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 2 [Note] WSREP: Synchronized with group, ready for connections
May 01 14:34:59 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:34:59 2 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
May 01 14:34:59 mysql-backend-03 /etc/mysql/debian-start[7044]: Checking for insecure root accounts.
May 01 14:34:59 mysql-backend-03 /etc/mysql/debian-start[7048]: Triggering myisam-recover for all MyISAM tables and aria-recover for all Ari>
May 01 14:35:01 mysql-backend-03 mariadbd[6743]: 2025-05-01 14:35:01 0 [Note] WSREP: (74b588d4-8ee7, 'ssl://192.168.56.23:4567') turning mes>
```

Для проверки состояния кластера воспользуемся статьёй [Using Status Variables](https://galeracluster.com/library/documentation/monitoring-cluster.html). Проверим, указанные в статье переменные кластера:

```text
❯ vagrant ssh mysql-backend-01 -c "sudo mariadb -e \"SHOW GLOBAL STATUS WHERE Variable_name in ('wsrep_cluster_state_uuid', 'wsrep_cluster_conf_id', 'wsrep_cluster_size', 'wsrep_cluster_status', 'wsrep_ready', 'wsrep_connected', 'wsrep_local_state_comment', 'wsrep_local_recv_queue_avg', 'wsrep_flow_control_paused', 'wsrep_cert_deps_distance', 'wsrep_local_send_queue_avg');\""
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_send_queue_avg | 0.000234852                          |
| wsrep_local_recv_queue_avg | 0.0208333                            |
| wsrep_flow_control_paused  | 4.25133e-05                          |
| wsrep_cert_deps_distance   | 56.4852                              |
| wsrep_local_state_comment  | Synced                               |
| wsrep_cluster_conf_id      | 8                                    |
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_state_uuid   | 4825d4e1-2699-11f0-8e08-87e21ae24a04 |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+

❯ vagrant ssh mysql-backend-02 -c "sudo mariadb -e \"SHOW GLOBAL STATUS WHERE Variable_name in ('wsrep_cluster_state_uuid', 'wsrep_cluster_conf_id', 'wsrep_cluster_size', 'wsrep_cluster_status', 'wsrep_ready', 'wsrep_connected', 'wsrep_local_state_comment', 'wsrep_local_recv_queue_avg', 'wsrep_flow_control_paused', 'wsrep_cert_deps_distance', 'wsrep_local_send_queue_avg');\""
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_send_queue_avg | 0                                    |
| wsrep_local_recv_queue_avg | 0.745404                             |
| wsrep_flow_control_paused  | 4.20726e-05                          |
| wsrep_cert_deps_distance   | 56.4852                              |
| wsrep_local_state_comment  | Synced                               |
| wsrep_cluster_conf_id      | 8                                    |
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_state_uuid   | 4825d4e1-2699-11f0-8e08-87e21ae24a04 |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+

❯ vagrant ssh mysql-backend-03 -c "sudo mariadb -e \"SHOW GLOBAL STATUS WHERE Variable_name in ('wsrep_cluster_state_uuid', 'wsrep_cluster_conf_id', 'wsrep_cluster_size', 'wsrep_cluster_status', 'wsrep_ready', 'wsrep_connected', 'wsrep_local_state_comment', 'wsrep_local_recv_queue_avg', 'wsrep_flow_control_paused', 'wsrep_cert_deps_distance', 'wsrep_local_send_queue_avg');\""
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_send_queue_avg | 0                                    |
| wsrep_local_recv_queue_avg | 1.02298                              |
| wsrep_flow_control_paused  | 3.44676e-05                          |
| wsrep_cert_deps_distance   | 56.4852                              |
| wsrep_local_state_comment  | Synced                               |
| wsrep_cluster_conf_id      | 8                                    |
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_state_uuid   | 4825d4e1-2699-11f0-8e08-87e21ae24a04 |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
```

Как видно кластер полностью исправен.

Для проверки отказоустойчивости можно выполнить следующее:

1. Зайдём на [https://192.168.56.51](https://192.168.56.51) - узел **mysql-backend-01** (имя пользователя **glpi**, пароль **glpi**).
2. Создадим заявку (**Поддержка** -> **Заявки** -> **Добавить**).
3. Выключим **mysql-backend-01**.

**IP** адрес переехал **mysql-backend-03**, никакие данные не были потеряны и заявка осталась в системе:

[!Тестовая заяка 1](images/task.png)
