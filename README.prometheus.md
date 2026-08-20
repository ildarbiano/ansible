
# ssl
ansible monigen -m shell -a "mkdir -p /opt/nginx/ssl && openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /opt/nginx/ssl/server.key \
-out /opt/nginx/ssl/server.crt \
-subj '/C=RU/ST=Moscow/L=Moscow/O=Dev/CN=192.168.0.44'" \
--vault-password-file ~/.ansible_vault_pass
# ping
ansible monigen -m ping -i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# создадим структуру ролей для мониторинга:
cd ~/ansible/moex_k8s/roles
mkdir -p node_exporter/{tasks,templates,vars,handlers,defaults}
mkdir -p postgres_exporter/{tasks,templates,vars,handlers,defaults}
mkdir -p prometheus/{tasks,templates,vars,handlers,defaults}
mkdir -p roles/nginx_exporter/{tasks,templates,vars,handlers,defaults}
mkdir -p roles/jmx_exporter/{tasks,templates,vars,handlers,defaults}
mkdir -p docker_setup/{tasks,templates,vars,handlers,defaults}
# Создадим playbook для 
touch playbooks/docker-setup.yml
# Создаем roles/docker_setup/tasks/main.yml:
cat > roles/docker_setup/tasks/main.yml << 'EOF'
# Создадим роль для exporter:
cat > roles/node_exporter/tasks/main.yml << 'EOF'
cat > roles/postgres_exporter/tasks/main.yml << 'EOF'
touch roles/nginx_exporter/tasks/main.yml
touch roles/jmx_exporter/tasks/main.yml
# Создадим playbook для деплоя мониторинга:
cat > playbooks/prometheus-deploy.yml << 'EOF'

##### запусти роль docker_setup напрямую:
ansible monigen -m include_role -a name=docker_setup \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Запусти docker-setup только для monigen:
ansible-playbook playbooks/docker-setup.yml \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass \
--limit monigen
# Проверь, что Docker доступен на хосте:
ansible monigen -m shell -a "docker --version" \
--vault-password-file ~/.ansible_vault_pass
# 
ansible-playbook playbooks/prometheus-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
# # На хосте k8s проверь, есть ли образ
ansible k8s -m shell -a "docker images | grep nginx-prometheus" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
###### Проверяем:
# Проверяем k8s
curl http://192.168.0.55:9100/metrics | head -10
# Tomcat
curl http://192.168.0.55:8080
# JMX Exporter
curl http://192.168.0.55:9010/metrics | head -10
# Проверяем pgs
curl http://192.168.0.66:9100/metrics | head -10
# Проверяем metrics
curl http://192.168.0.44:9100/metrics | head -10
# Проверяем Postgres Exporter на pgs:
curl http://192.168.0.66:9187/metrics | head -10
# Node Exporter (должен работать)
curl -s http://192.168.0.55:9100/metrics | head -5
# Nginx Exporter
curl -s http://192.168.0.55:9113/metrics | head -5
# JMX Exporter (Tomcat)
curl -s http://192.168.0.55:9010/metrics | head -5
# Проверь Postgres Exporter на pgs:
curl -s http://192.168.0.66:9187/metrics | head -5
#  Проверяем статус контейнера на pgs:
ansible pgs -m shell -a "docker ps -a | grep postgres-exporter" \
--vault-password-file ~/.ansible_vault_pass
# Смотрим логи контейнера:
ansible pgs -m shell -a "docker logs postgres-exporter --tail 30" \
--vault-password-file ~/.ansible_vault_pass
# Проверяем, какие переменные определены на pgs:
ansible pgs -m shell -a "env | grep -E 'postgres|vault'" \
--vault-password-file ~/.ansible_vault_pass
# Проверяем, что PostgreSQL доступен с pgs:
ansible pgs -m shell -a "docker exec postgres psql \
-U appuser \
-d appdb \
-c 'SELECT 1'" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверяем, какой пользователь есть в PostgreSQL:
ansible pgs -m shell -a "docker exec postgres psql \
-U postgres \
-c '\du'" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверяем переменные в vault:
ansible-vault view group_vars/dev/vault.yml 
--vault-password-file ~/.ansible_vault_pass
# Проверяем, как запущен PostgreSQL:
ansible pgs -m shell -a "docker inspect postgres | grep -A 5 Env" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass

###### переходим к Prometheus и Grafana на monigen
# Создаем roles/prometheus/tasks/main.yml:
cat > roles/prometheus/tasks/main.yml << 'EOF'
# Создаем roles/prometheus/templates/prometheus.yml.j2:
cat > roles/prometheus/templates/prometheus.yml.j2 << 'EOF'
# Создаем roles/prometheus/handlers/main.yml:
cat > roles/prometheus/handlers/main.yml << 'EOF'
# Запускаем:
ansible-playbook playbooks/prometheus-deploy.yml \
--vault-password-file ~/.ansible_vault_pass

##### Проверяем:
# Prometheus
curl http://192.168.0.44:9090/api/v1/query?query=up
# статус targets в Prometheus:
# Открой в браузере: 
http://192.168.0.44:9090/targets
Или через curl:
curl -s http://192.168.0.44:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health, error: .lastError}'
# Проверяем статус контейнеров на monigen:
ansible monigen -m shell \
-a "docker ps -a" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Смотрим логи Prometheus:
ansible monigen -m shell -a "docker logs prometheus --tail 30" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Смотрим логи Grafana:
ansible monigen -m shell -a "docker logs grafana --tail 30" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверяем, что контейнеры запустились:
ansible monigen -m shell -a "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass


nginx-exporter
jmx_exporter

Nginx:            Используй nginx-exporter для метрик upstream‑серверов (время ответа бэкенда, количество 5xx ошибок).
Java (Tomcat):    jmx_exporter для метрик JVM (GC pauses, heap usage).
Настрой k6 на экспорт метрик в Prometheus. В docker-compose.yml для сервиса k6 добавь:
command: run /scripts/load-test.js --out experimental-prometheus-rw=http://prometheus:9090/api/v1/write
Это позволит Grafana отображать метрики k6 в реальном времени.
Лимитируй ресурсы в Docker. Для сервиса k6 в docker-compose.yml обязательно укажи:
yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 2G
Это гарантирует, что даже при пиковой нагрузке k6 не «положит» Prometheus и Grafana, и ты сможешь увидеть метрики самого сбоя.
Итог: Размещение k6, Prometheus и Grafana на одном хосте monigen — это стандартный и рабочий подход для тестовых стендов. Главное — не забывать про лимиты ресурсов для k6 и правильно настроить экспорт его собственных метрик в Prometheus, чтобы видеть полную картину, а не только загрузку железа.

