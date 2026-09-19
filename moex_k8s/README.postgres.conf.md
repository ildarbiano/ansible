########## Создаем playbooks/postgres-inspect.yml
# Запускаем инспекцию
bash
ansible-playbook playbooks/postgres-inspect.yml \
--vault-password-file ~/.ansible_vault_pass -v

########## переход на шаблон создания БД
# шаблон init-db.sql.j2 (новый файл)
cat > roles/postgres/templates/init-db.sql.j2 << 'EOF'
# Меняем задачу в tasks/main.yml
- name: Create init SQL script (create database and user)
# Запускаем
bash
ansible-playbook playbooks/postgres-deploy.yml \
--vault-password-file ~/.ansible_vault_pass -v

Итог
✅ В контейнере PostgreSQL можно создать несколько баз данных.
✅ init-db.sql выполняется только один раз — при первом запуске.
✅ Для добавления новых баз после запуска используй SQL-команды.

######## Создаем playbook для управления базами данных
cat > playbooks/postgres-manage-dbs.yml << 'EOF'
---
# Запускаем
bash
ansible-playbook playbooks/postgres-manage-dbs.yml \
--vault-password-file ~/.ansible_vault_pass -v

######## Создаем playbook для удаления баз данных
cat > playbooks/postgres-delete-dbs.yml << 'EOF'
---
# добавляем явный список в host_vars/pgs/vars.yml
postgres_dbs_to_delete:

# Запуск в режиме "проверки" (dry-run) — безопасно
По умолчанию postgres_dry_run: true, поэтому ничего не удаляется, только показывается, что было бы удалено.
cat playbooks/postgres-delete-dbs.yml | grep postgres_dry_run
      when: postgres_dry_run | default(true) | bool
      when: not (postgres_dry_run | default(true) | bool)
      when: not (postgres_dry_run | default(true) | bool) and item.changed

bash
ansible-playbook playbooks/postgres-delete-dbs.yml --vault-password-file ~/.ansible_vault_pass -v

# выполнить реально удаление:
ansible-playbook playbooks/postgres-delete-dbs.yml \
--vault-password-file ~/.ansible_vault_pass \
-e "postgres_dry_run=false" -v