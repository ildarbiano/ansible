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
# запуск
ansible-playbook playbooks/postgres-check-connect-throw-k8s.yml \
--vault-password-file ~/.ansible_vault_pass -v


#### =====================
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
curl -k https://192.168.0.55/api/data

