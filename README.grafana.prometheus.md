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