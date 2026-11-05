# Роль opensearch

Ставит OpenSearch и OpenSearch Dashboards (single-node) из официальных tar.gz на Ubuntu 22.04/24.04.

Обязательный параметр: `opensearch_admin_password` (передавайте через Ansible Vault).
Все параметры и их значения по умолчанию: `defaults/main.yml`. Подробности — в корневом README репозитория.

Пример:

    - hosts: opensearch
      become: true
      roles:
        - opensearch
