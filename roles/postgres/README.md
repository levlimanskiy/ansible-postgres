# Роль: postgres

Устанавливает PostgreSQL из стандартных репозиториев Ubuntu, инициализирует
кластер в произвольном каталоге данных, управляет `postgresql.conf` и
`pg_hba.conf`, запускает PostgreSQL под собственной systemd-службой, создаёт
базы данных и пользователей и устанавливает postgres-exporter как systemd-службу.

## Требования

- Целевой хост: Ubuntu с systemd и доступом к sudo.
- Контроллер: Ansible и коллекция `community.postgresql`:
  `ansible-galaxy collection install -r requirements.yml`
- Контроллеру нужен доступ в интернет для скачивания релиза экспортера.

## Переменные

Заданы в `defaults/main.yml`. Любую из них можно переопределить.

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `postgres__packages` | `[postgresql, python3-psycopg2, acl]` | Устанавливаемые пакеты |
| `postgres__data_dir` | `/data/postgres/data` | Каталог данных кластера (`$PGDATA`) |
| `postgres__opts` | `{}` | Словарь параметров `postgresql.conf`. Строковые значения нужно заключать во внутренние кавычки, например `"'localhost'"` |
| `postgres__hba` | локальные правила peer + `127.0.0.1/32` scram-sha-256 | Список правил `pg_hba.conf` (ключи: `type`, `database`, `user`, `address` (необязательный), `method`) |
| `postgres__databases` | `[]` | Список словарей: `name` (обязательный), `encoding`, `state` (необязательные) |
| `postgres__users` | `[]` | Список словарей: `name`, `password` |
| `postgres__exporter_version` | `0.20.1` | Версия релиза postgres-exporter |
| `postgres__exporter_port` | `9187` | Порт экспортера (слушает на 127.0.0.1) |
| `postgres__exporter_db_user` | `postgres_exporter` | Роль БД для экспортера (входит в `pg_monitor`) |
| `postgres__exporter_db_password` | нет, обязательная | Пароль этой роли. Задаётся в инвентаре, значение берётся из vault |

Внутренние переменные (префикс `__`) лежат в `vars/main.yml` и не предназначены
для изменения.

## Секреты

Пароли не хранятся в открытом виде. Они определяются в файле, зашифрованном
`ansible-vault` (`inventories/main/group_vars/all/vault.yml`), а в обычном файле
переменных на них даются ссылки:

```yaml
postgres__users:
  - name: app
    password: "{{ vault_app_password }}"
postgres__exporter_db_password: "{{ vault_exporter_password }}"
```

## Пример плейбука

```yaml
- name: Install PostgreSQL
  hosts: vm1
  become: true
  roles:
    - postgres
```

с, например, такими значениями в `group_vars`:

```yaml
postgres__opts:
  listen_addresses: "'localhost'"
  max_connections: 150

postgres__databases:
  - name: appdb
  - name: analytics
    encoding: UTF8
```

## Примечания

- Служба кластера Ubuntu по умолчанию (`postgresql@<версия>-main`) останавливается
  и маскируется, так как роль запускает собственную службу (`postgres-custom`)
  с нестандартным каталогом данных.
- Изменение `postgresql.conf` перезапускает PostgreSQL, изменение `pg_hba.conf`
  перечитывает конфигурацию (через хендлеры).


