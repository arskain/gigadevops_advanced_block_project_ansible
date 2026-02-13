# HA PostgreSQL Cluster с Patroni + etcd + Ansible

Высокодоступный кластер PostgreSQL на базе Patroni, etcd и Ansible.

## Структура проекта
ha-cluster/
├── ansible.cfg               # Основные настройки Ansible (inventory, become, etc.)
├── group_vars/
│   └── cluster.yml           # Общие переменные: пароли, IP-адреса, версии, имена
├── inventories/
│   └── hosts                 # Список нод (группа [cluster])
├── roles/
│   ├── etcd/                 # Установка и настройка etcd-кластера
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── etcd.conf.j2
│   │   └── handlers/
│   ├── postgresql/           # Установка PostgreSQL (без запуска дефолтного сервиса)
│   │   └── tasks/
│   │       └── main.yml
│   └── patroni/              # Установка Patroni, конфиг, запуск, сброс кластера
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   └── patroni.yml.j2
│       └── handlers/
└── playbooks/
└── site.yml              # Главный плейбук (деплой всего кластера)


## Назначение основных файлов

| Файл/Директория                      | За что отвечает                                                                 |
|--------------------------------------|---------------------------------------------------------------------------------|
| `ansible.cfg`                        | Глобальные настройки Ansible (inventory, become, пути, timeout и т.д.)         |
| `group_vars/cluster.yml`             | Переменные кластера: IP-адреса, пароли, версии PostgreSQL, имя пользователя replication |
| `inventories/hosts`                  | Список нод (группа cluster: node1, node2, node3)                                |
| `roles/etcd/`                        | Установка и настройка etcd (консенсус и DCS для Patroni)                        |
| `roles/postgresql/`                  | Установка пакетов PostgreSQL (Patroni сам будет управлять экземплярами)         |
| `roles/patroni/`                     | Установка Patroni, генерация конфига, запуск сервиса, сброс кластера при необходимости |
| `roles/patroni/templates/patroni.yml.j2` | Шаблон основного конфига Patroni (scope, etcd, bootstrap, pg_hba и т.д.)        |
| `playbooks/site.yml`                 | Главный плейбук — последовательное развёртывание всего кластера                 |

## Как запускать

### 1. Полный деплой с нуля (сброс кластера)

```bash
ansible-playbook playbooks/site.yml --ask-become-pass -e reset_cluster=true

- Сбрасывает состояние Patroni
- Очищает данные PostgreSQL
- Удаляет ключи в etcd
- Деплоит всё заново

### 2. Обычный деплой / обновление конфигов (без сброса)

ansible-playbook playbooks/site.yml --ask-become-pass

### 3. Только роль Patroni (например, после правки шаблона)

ansible-playbook playbooks/site.yml --ask-become-pass --tags patroni_setup

### 4. Только сброс (если кластер в deadlock)

ansible-playbook playbooks/site.yml --ask-become-pass -e reset_cluster=true --tags reset_cluster

## Проверки после деплоя

### 1. Статус кластера (с любой ноды)

patronictl -c /etc/patroni/config.yml list

Ожидаемый результат:
Member | Host         | Role    | State     | TL | Lag in MB
node1  | 10.10.10.10  | Leader  | running   | X  | 
node2  | 10.10.10.30  | Replica | streaming | X  | 0
node3  | 10.10.10.40  | Replica | streaming | X  | 0

### 2. Проверка репликации

На текущем Leader:
psql -h 10.10.10.10 -U postgres -W

```sql
CREATE TABLE test(id int);
INSERT INTO test VALUES (1);
```

Подключится к Replica:

```bash
psql -h 10.10.10.30 -U postgres -W
```

```sql
SELECT * FROM test;
```

### 3. Логи Patroni (на любой ноде)

sudo journalctl -u patroni -f

### 4. Тест failover

На лидере:
`sudo pkill patroni`

Через 10–20 секунд снова:
`patronictl -c /etc/patroni.yml list`

