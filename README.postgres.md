cd ~/ansible/moex_k8s
mkdir -p roles/postgres/{tasks,templates,handlers,vars}
cat > roles/postgres/tasks/main.yml << 'EOF'
---
EOF
cat > playbooks/postgres-deploy.yml << 'EOF'
---
EOF
# Расшифровываем vault
ansible-vault decrypt group_vars/dev/vault.yml --vault-password-file ~/.ansible_vault_pass
# Добавляем пароль PostgreSQL
cat >> group_vars/dev/vault.yml << 'EOF'

# PostgreSQL password
vault_postgres_password: "654321"
EOF
# Снова шифруем
ansible-vault encrypt group_vars/dev/vault.yml --vault-password-file ~/.ansible_vault_pass
# Шаг 5. Запускаем развертывание PostgreSQL
bash
ansible-playbook playbooks/postgres-deploy.yml \
--vault-password-file ~/.ansible_vault_pass -v

#### ======== проверка - починка ============
# На хосте pgs
ssh ilya@192.168.0.66

# Проверить, какой процесс на порту 5432
sudo ss -tlnp | grep 5432
# Проверить, установлен ли системный PostgreSQL
dpkg -l | grep postgresql
# Остановить системный PostgreSQL
sudo systemctl stop postgresql
sudo systemctl disable postgresql
# сценарий проверки
cat > playbooks/postgres-check-connect-throw-k8s.yml << 'EOF'
---
# запуск
ansible-playbook playbooks/postgres-check-connect-throw-k8s.yml \
--vault-password-file ~/.ansible_vault_pass -v


#### ====== API ==========
### через Application POST
curl -k -X POST https://192.168.0.55/api/data \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "value": "test_data_1", "timestamp": "2026-08-31T12:00:00"}'
### через Application GET
ansible pgs -m shell \
-a "docker exec postgres psql \
-U ilya-ansible \
-d dtbase_1 \
-c 'SELECT id, method, data, response_time_ms FROM first_pastman_req ORDER BY id DESC LIMIT 5;'" \
--vault-password-file ~/.ansible_vault_pass
# посчитать количество записей
curl -k https://192.168.0.55/api/data/count
# Получи все записи
curl -k https://192.168.0.55/api/data
curl -k https://192.168.0.55/api/data | jq .
  # jq — это JSON-процессор для командной строки. Он позволяет:
    Красиво форматировать JSON
    Фильтровать данные
    Извлекать конкретные поля
    Строить сложные запросы
  # | (пайп)	Передаёт вывод curl в jq
jq .	          Красивое форматирование
jq '.[]'	      Развернуть массив
jq '.[].id'	    Выбрать поле id
jq '.[0].data'  Получить только data из первой записи
jq '.[-1]'	    Последняя запись
jq 'length'	    Количество записей
jq '.[] | select(.id > 5)'	      Фильтр
jq '.[] | {id, responseTimeMs}'   Получить только id и responseTimeMs


## ======= docker ======
# Войти внутрь контейнера
docker exec -it postgres bash
# Посмотреть все запущенные контейнеры на хосте k8s:
sudo docker ps    
# Посмотреть логи контейнера
docker logs postgres --tail 20
# Вывод всех контейнеров с сетями
sudo docker ps --format 'table {{.Names}}\t{{.Networks}}'
# Память внутри контейнера PostgreSQL:
free -h

# ====== psql =================
# Запуск psql внутри контейнера:
psql -U ilya-ansible -d dtbase_1
# Посмотрим структуру таблицы "first_pastman_req"
dtbase_1=# \d public.first_pastman_req
# Размер всей базы dtbase_1:
dtbase_1=# SELECT pg_database_size('dtbase_1') / 1024 / 1024 AS size_mb;
# Размер таблицы first_pastman_req:
dtbase_1=# SELECT pg_total_relation_size('first_pastman_req') / 1024 / 1024 AS table_size_mb;
# Подробно о размере таблицы (таблица + индексы):
dtbase_1=# SELECT
pg_size_pretty(pg_table_size('first_pastman_req')) AS table_size,
pg_size_pretty(pg_indexes_size('first_pastman_req')) AS index_size,
pg_size_pretty(pg_total_relation_size('first_pastman_req')) AS total_size;
# Количество записей в таблице:
dtbase_1=# SELECT COUNT(*) FROM first_pastman_req;

