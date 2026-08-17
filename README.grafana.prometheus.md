# добавить
mkdir host_vars/monigen
inventories/dev/hosts.yml
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
mkdir -p grafana/{tasks,templates,vars,handlers,defaults}
mkdir -p docker_setup/{tasks,templates,vars,handlers,defaults}
# Создадим playbook для 
touch playbooks/docker-setup.yml
# Создаем roles/docker_setup/tasks/main.yml:
cat > roles/docker_setup/tasks/main.yml << 'EOF'
# Создадим роль node_exporter:
cat > roles/node_exporter/tasks/main.yml << 'EOF'
# Создадим роль postgres_exporter:
cat > roles/postgres_exporter/tasks/main.yml << 'EOF'
# Создадим playbook для деплоя мониторинга:
cat > playbooks/monitoring-deploy.yml << 'EOF'

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
ansible-playbook playbooks/monitoring-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
#
ansible-playbook playbooks/monitoring-deploy.yml \
--vault-password-file ~/.ansible_vault_pass \
--limit pgs

###### Проверяем Node Exporter на всех хостах:
# Проверяем k8s
curl http://192.168.0.55:9100/metrics | head -10
# Проверяем pgs
curl http://192.168.0.66:9100/metrics | head -10
# Проверяем monigen
curl http://192.168.0.44:9100/metrics | head -10
# Проверяем Postgres Exporter на pgs:
curl http://192.168.0.66:9187/metrics | head -10
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
# Создаем roles/grafana/tasks/main.yml:
cat > roles/grafana/tasks/main.yml << 'EOF'
# Обновляем playbooks/monitoring-deploy.yml:
cat > playbooks/monitoring-deploy.yml << 'EOF'
# Создаем roles/prometheus/handlers/main.yml:
cat > roles/prometheus/handlers/main.yml << 'EOF'
# Запускаем:
ansible-playbook playbooks/monitoring-deploy.yml \
--vault-password-file ~/.ansible_vault_pass

##### Проверяем:
# Prometheus
curl http://192.168.0.44:9090/api/v1/query?query=up
# Grafana
curl http://192.168.0.44:3000
##### Если Grafana запустилась:
# Открой в браузере:
# Grafana: 
http://192.168.0.44:3000 
# (логин: admin, пароль: admin)
# Prometheus: 
http://192.168.0.44:9090
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

#### Grafana 
# 1
Всё, что будет сохранено в /var/lib/grafana внутри контейнера, на самом деле будет лежать в папке /opt/grafana/data на хосте monigen (192.168.0.44).
# 2 Второй подход: "Дашборды как код" (более правильный)
Это промышленный подход, который решает вашу задачу по переносу на другую инфраструктуру . Идея в том, чтобы хранить дашборды не в базе данных Grafana, а в виде JSON-файлов вместе с вашим Ansible-кодом. При развертывании Ansible сам "скормит" их Grafana .

Как это будет выглядеть в вашей архитектуре
У вас будет структура в Ansible-проекте:
text
~/ansible/moex_k8s/
├── roles/
│   └── grafana/
│       ├── tasks/
│       │   └── main.yml
│       └── files/                         # <-- Новая папка
│           └── dashboards/                # <-- Для JSON-файлов дашбордов
│               ├── node-exporter-full.json
│               └── postgres-dashboard.json
Вам нужно добавить в roles/grafana/tasks/main.yml блок для копирования папки с JSON-дашбордами и блок для настройки "провайдера" дашбордов 
# Как сохранить созданные дашборды
Пока у вас нет готовых дашбордов, можно:
    1\Экспортировать через интерфейс: Зайти в дашборд → шестеренка → Share → вкладка Export → Save to file .
    2\Положить полученный JSON-файл в папку roles/grafana/files/dashboards/ в вашем Ansible-проекте.
    3\При следующем запуске плейбука этот дашборд появится автоматически.

# Как защитить дашборды от случайной потери:
# Сделай бэкап данных Grafana
ansible monigen -m shell -a "tar -czf /tmp/grafana-backup-$(date +%Y%m%d).tar.gz /opt/grafana/data" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Скопируй бэкап на ansible-master
scp ilya@192.168.0.44:/tmp/grafana-backup-*.tar.gz ~/backups/


Nginx: Используй nginx-exporter для метрик upstream‑серверов (время ответа бэкенда, количество 5xx ошибок).
Java (Tomcat): jmx_exporter для метрик JVM (GC pauses, heap usage).
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