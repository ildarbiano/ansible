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

============== проверка - починка ============
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

EOF
# запуск
ansible-playbook playbooks/postgres-check-connect-throw-k8s.yml \
--vault-password-file ~/.ansible_vault_pass -v


======== УПРАВЛЕНИЕ И НАСТРОЙКА БД:
sudo cat /opt/postgres/init/init-db.sql
CREATE USER ilya-ansible WITH PASSWORD '654321';
CREATE DATABASE dtbase_1 OWNER ilya-ansible;
GRANT ALL PRIVILEGES ON DATABASE dtbase_1 TO ilya-ansible;

======== psql ======================================================================================
# Учитывая, что у тебя хост называется postgres-docker (и ты строишь стенд в виртуалках/контейнерах), 
# Как быстро проверить, где вообще есть psql
which psql
find / -name psql 2>/dev/null
# список файлов, где есть упоминание
find ~/ansible/moex_k8s -type f \
-name "*.yml" -o -name "*.yaml" -o -name "*.j2" | xargs grep -l "app-net" 2>/dev/null
# список файлов, где есть упоминание, но с выводом строки - удобней и наглядней
grep -r \
"app-net" ~/ansible/moex_k8s --include="*.yml" --include="*.yaml" --include="*.j2" 2>/dev/null
# варианты, как быстро получить psql
apt-get update
apt-get install -y postgresql-client
# Вариант 2: использовать psql из самого контейнера с сервером
# Если у тебя контейнер с PostgreSQL уже запущен, можно выполнить psql внутри него:
docker exec -it <имя_контейнера> psql -h localhost -U ilya-ansible
# Если хочешь работать прямо из контейнера (без установки клиента)
# Можно вообще не использовать внешний psql, а зайти внутрь контейнера и работать там:
docker exec -it postgres psql -U ilya-ansible   # Имя postgres (как алиас контейнера) работает только внутри сети Docker
docker exec -t postgres psql -U ilya-ansible -d dtbase_1
# -c "\dt" - просмотр таблиц внутри базы
docker exec -t postgres psql -U ilya-ansible -d dtbase_1 -c "\dt"
# проверить, какие базы реально существуют в контейнере
docker exec -t postgres psql -U ilya-ansible -c "\l"
docker exec -t postgres psql -U postgres -c "\l"
#  посмотреть список нужных баз (самый надёжный)
docker exec -t postgres psql -U ilya-ansible -d dtbase_1 -c "SELECT datname FROM pg_database WHERE datname IN ('dtbase_1','base_2','dtbase_logs');"
# или сразу как суперпользователь:
docker exec -it postgres psql -U postgres
# узнать имя контейнера:
docker ps
# Быстрый способ получить IP контейнера
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' postgres
## Важно: порт -p 5434 — это порт на хосте. Внутри контейнера Postgres слушает 5432. При подключении с хоста указываем именно проброшенный порт (5434).
psql -h 172.19.0.3 -p 5432 -U ilya-ansible  
#
psql -h postgres -p 5434 -U ilya-ansible
psql -h <IP_контейнера_или_хоста> -p <порт> -U ilya-ansible
#  посмотри переменные окружения контейнера:
docker exec postgres env | grep POSTGRES




