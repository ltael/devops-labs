# Лабораторная работа №1 — Ansible-роль для OpenSearch

## Цель

Разработать Ansible-роль для установки и настройки OpenSearch и OpenSearch Dashboards 3.8
в режиме single-node на удалённой Linux-машине: установка выбранной версии, настройка системы
и сервиса, запуск, автоматическое применение изменений конфигурации и проверка готовности через API.

## Окружение

| Компонент | Версия / значение |
|---|---|
| Управляющая машина | ноутбук, Ubuntu 24.04.4 LTS |
| VirtualBox | 7.2.22 |
| Vagrant | 2.4.9 |
| Ansible | ansible-core 2.21.5 |
| Python | 3.12.3 |
| Целевая машина | ВМ Ubuntu 24.04 (образ `bento/ubuntu-24.04`), 2 CPU, 4 ГБ RAM, IP `192.168.56.10` |
| OpenSearch / OpenSearch Dashboards | 3.8.0 |

Целевая машина — виртуальная машина VirtualBox, созданная через Vagrant (`Vagrantfile` в корне репозитория).
Ansible подключается к ней по SSH, как к удалённому серверу. Docker и другие контейнеры не используются.

## Структура репозитория

```
Vagrantfile                      описание тестовой ВМ
ansible.cfg                      настройки Ansible (inventory, roles_path)
.yamllint                        правила yamllint, совместимые с ansible-lint
playbook.yml                     вызывает роль opensearch на группе хостов opensearch
inventory/
  hosts.yml                      список хостов
  group_vars/opensearch/
    main.yml                     связывает пароль из Vault с переменной роли
    vault.yml                    пароль admin, зашифрован Ansible Vault
roles/opensearch/                роль
docs/                            выводы команд и скриншоты для отчёта
```

## Роль opensearch

Сторонние роли не используются. Установка выполняется из официальных архивов tar.gz
(в них уже есть JDK для OpenSearch и Node.js для Dashboards).

| Директория | Содержимое |
|---|---|
| `defaults/` | параметры, которые пользователь может переопределить |
| `vars/` | внутренние значения: пути установки, ссылки на архивы, список пакетов, архитектура |
| `tasks/` | `main.yml` и шаги: `preflight`, `system`, `install`, `config`, `service`, `healthcheck`, `dashboards` |
| `handlers/` | применение sysctl, `daemon-reload`, перезапуск OpenSearch и Dashboards |
| `templates/` | `opensearch.yml`, `heap.options`, `opensearch_dashboards.yml`, unit-файлы systemd обеих служб |
| `files/` | `99-opensearch.conf` с `vm.max_map_count` |
| `meta/` | описание роли, минимальная версия Ansible, поддерживаемые платформы |

### Как работает роль

1. **preflight** — проверяет, что задан пароль admin и корректный размер heap; без них роль останавливается сразу.
2. **system** — ставит зависимости, создаёт системного пользователя и группу `opensearch`,
   копирует `99-opensearch.conf` (`vm.max_map_count = 262144`) и сразу применяет его через handler
   (`flush_handlers`), до первого запуска OpenSearch.
3. **install** — создаёт каталоги данных и логов, скачивает архив с проверкой SHA-512, распаковывает его
   в `/opt/opensearch-<версия>` и делает ссылку `/opt/opensearch`. При смене версии новая распаковывается
   рядом, ссылка переключается, handler перезапускает службу. Данные хранятся отдельно в `/var/lib/opensearch`.
   Затем один раз выполняется демо-скрипт security-плагина: он создаёт TLS-сертификаты и задаёт пароль admin.
4. **config** — формирует `opensearch.yml` и `jvm.options.d/heap.options` из шаблонов Jinja2.
   При изменении любого из них срабатывает handler `Restart opensearch`.
5. **service** — устанавливает unit-файл systemd (лимиты `LimitNOFILE`, `LimitMEMLOCK`), выполняет отложенные
   handlers и включает службу с автозапуском (`enabled: true`). Благодаря `flush_handlers` перед `started`
   при первом запуске служба стартует один раз, без лишнего перезапуска.
6. **healthcheck** — проверяет готовность через `GET _cluster/health`: до 30 попыток с паузой 10 секунд,
   статус должен быть `green` или `yellow`. Если не дождались — роль завершается с ошибкой.
7. **dashboards** — аналогично ставит OpenSearch Dashboards под отдельным пользователем, формирует конфиг
   и unit-файл, включает автозапуск и проверяет `/api/status` с ограниченным числом попыток.

Повторный запуск роли ничего не меняет (`changed=0`) и не перезапускает службы.
Код проверен линтерами: `yamllint` — без замечаний, `ansible-lint` — `Passed` (профиль `production`).

## Параметры роли

Все параметры задаются в `roles/opensearch/defaults/main.yml` и переопределяются через inventory,
`group_vars` или `-e` без правки задач и шаблонов.

| Параметр | По умолчанию | Почему вынесен |
|---|---|---|
| `opensearch_version` | `3.8.0` | обновление версии без изменения задач |
| `opensearch_dashboards_version` | `3.8.0` | версия Dashboards должна совпадать с версией OpenSearch |
| `opensearch_install_root` | `/opt` | в разных организациях свои стандарты размещения ПО |
| `opensearch_data_dir` | `/var/lib/opensearch` | данные часто выносят на отдельный диск |
| `opensearch_log_dir` | `/var/log/opensearch` | логи могут собираться из другого каталога |
| `opensearch_user` / `opensearch_group` | `opensearch` | имя служебной учётной записи может быть регламентировано |
| `opensearch_dashboards_user` | `opensearch-dashboards` | то же для Dashboards |
| `opensearch_cluster_name` | `lab-cluster` | имя кластера различается между окружениями |
| `opensearch_node_name` | имя хоста из inventory | уникальное имя узла без ручной настройки |
| `opensearch_network_host` | `0.0.0.0` | интерфейс, на котором слушает API; в лабе нужен доступ с управляющей машины |
| `opensearch_http_port` | `9200` | порт может быть занят или стандартизирован иначе |
| `opensearch_heap_size` | `1g` | зависит от объёма памяти машины (рекомендуется не больше половины RAM) |
| `opensearch_memory_lock` | `true` | запрет вытеснения памяти JVM в swap; на некоторых системах отключают |
| `opensearch_extra_config` | `{}` | любые дополнительные настройки `opensearch.yml` без правки шаблона |
| `opensearch_admin_password` | пусто | секрет, задаётся только через Ansible Vault; пустое значение останавливает роль |
| `opensearch_dashboards_host` | `0.0.0.0` | интерфейс веб-интерфейса Dashboards |
| `opensearch_dashboards_port` | `5601` | порт может быть занят |
| `opensearch_dashboards_server_user` / `_password` | `kibanaserver` | служебная учётная запись Dashboards из демо-конфигурации |
| `opensearch_healthcheck_retries` | `30` | время старта зависит от мощности машины |
| `opensearch_healthcheck_delay` | `10` | пауза между попытками проверки, секунды |
| `opensearch_healthcheck_accepted_statuses` | `[green, yellow]` | `yellow` допустим для single-node, где реплики размещать негде |

## Безопасность

Пароль пользователя admin хранится в `inventory/group_vars/opensearch/vault.yml`, зашифрованном
Ansible Vault; пароль от Vault в репозиторий не добавлен. Задачи, работающие с паролем, помечены
`no_log: true`, поэтому пароль не попадает в вывод Ansible.

Используются демо-сертификаты security-плагина OpenSearch и стандартный пароль служебного
пользователя `kibanaserver`. Это допустимо для учебного стенда и будет заменено при настройке
безопасности в лабораторной работе №2.

## Запуск

```bash
vagrant up                                         # создать и запустить ВМ
ansible all -m ping                                # проверить доступ по SSH
ansible-vault create inventory/group_vars/opensearch/vault.yml
# внутри: vault_opensearch_admin_password: "<надёжный пароль>"
ansible-playbook playbook.yml --ask-vault-pass     # применить роль
```

## Результаты проверки

### Первый запуск на чистой ВМ

```
PLAY RECAP *********************************************************************
opensearch1                : ok=30   changed=23   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Полный вывод: [docs/run1.txt](docs/run1.txt).

### Повторный запуск — идемпотентность

```
PLAY RECAP *********************************************************************
opensearch1                : ok=23   changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

`changed=0`: роль ничего не меняет и не перезапускает службы. `skipped=2` — задачи распаковки архивов,
пропущенные благодаря `creates:`. Полный вывод: [docs/run2.txt](docs/run2.txt).

### GET _cluster/health

```bash
curl -sk -u admin "https://192.168.56.10:9200/_cluster/health?pretty"
```

```json
{
  "cluster_name" : "lab-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "discovered_master" : true,
  "discovered_cluster_manager" : true,
  "active_primary_shards" : 5,
  "active_shards" : 5,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Кластер в статусе `green`, один узел — режим single-node.

### Версия OpenSearch

```bash
curl -sk -u admin "https://192.168.56.10:9200/"
```

```json
{
  "name" : "opensearch1",
  "cluster_name" : "lab-cluster",
  "cluster_uuid" : "wwVMpJRGSLCyO7uydmwjRw",
  "version" : {
    "distribution" : "opensearch",
    "number" : "3.8.0",
    "build_type" : "tar",
    "build_hash" : "e5a3c5691be87af6c12dbe3e158c59c04ee72973",
    "build_date" : "2026-08-01T18:48:00.452816924Z",
    "build_snapshot" : false,
    "lucene_version" : "10.5.0",
    "minimum_wire_compatibility_version" : "2.19.0",
    "minimum_index_compatibility_version" : "2.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```

Установлена заданная версия `3.8.0` из архива (`build_type: tar`).

### Handlers: перезапуск только при изменении конфигурации

```bash
ansible-playbook playbook.yml --ask-vault-pass -e opensearch_heap_size=1536m
```

```
TASK [opensearch : Render JVM heap options] ************************************
changed: [opensearch1]
--
RUNNING HANDLER [opensearch : Restart opensearch] ******************************
changed: [opensearch1]
--
PLAY RECAP *********************************************************************
opensearch1                : ok=24   changed=2    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

Изменился только параметр heap: изменилась одна задача, сработал один handler перезапуска OpenSearch,
Dashboards не перезапускался (`changed=2` — шаблон и handler). Полный вывод: [docs/run-handler.txt](docs/run-handler.txt).

### Автозапуск после перезагрузки ВМ

```bash
vagrant ssh -c "systemctl is-enabled opensearch opensearch-dashboards"
```

```
enabled
enabled
```

После перезагрузки ВМ (`vagrant reload`) без запуска Ansible:

```json
{
  "cluster_name" : "lab-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "discovered_master" : true,
  "discovered_cluster_manager" : true,
  "active_primary_shards" : 5,
  "active_shards" : 5,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Обе службы включены в автозагрузку, после перезагрузки кластер поднялся сам и находится в статусе `green`.

### OpenSearch Dashboards

`GET _cluster/health` в Dev Tools (`http://192.168.56.10:5601`, вход под пользователем admin):

![Dev Tools: GET _cluster/health](docs/screenshots/devtools-health.png)

Главная страница Dashboards:

![OpenSearch Dashboards](docs/screenshots/dashboards.png)

## Выводы

Разработана Ansible-роль без сторонних ролей, которая устанавливает OpenSearch и OpenSearch Dashboards 3.8.0
в режиме single-node из официальных архивов, настраивает систему (параметры ядра, лимиты, служебные пользователи)
и службы systemd. Проверено: кластер в статусе `green`, роль идемпотентна (`changed=0` при повторном запуске),
службы перезапускаются только при изменении конфигурации, стартуют автоматически после перезагрузки,
а готовность проверяется через API с ограниченным числом попыток. В лабораторной работе №2 роль будет
расширена настройкой безопасности через Security REST API и автотестами Molecule.
