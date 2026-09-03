======== json ============
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


======== sql ================
# УПРАВЛЕНИЕ И НАСТРОЙКА БД:
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