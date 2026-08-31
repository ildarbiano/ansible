# Только статический IP
ansible-playbook site_static_IP.yml \
--vault-password-file ~/.ansible_vault_pass

# Полный деплой всего стенда
ansible-playbook site_full_install.yml \
-i inventories/dev/hosts.yml 
--vault-password-file ~/.ansible_vault_pass

# ========== SSL + vault ================
# проблема с измененным fingerprint. Выполни команду, которую рекомендует SSH:
ssh-keygen -f "/home/ilya/.ssh/known_hosts" -R "192.168.0.44"
# Затем попробуй подключиться снова:
ssh ilya@192.168.0.44
Если запросит подтверждение — введи yes.
# После этого проверь через Ansible:
ansible gnld -m ping -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
#  Проверка вручную (если есть доступ)
# Зайди на управляемый хост
ssh ilya@192.168.0.44
# Проверь vault
ansible-vault view group_vars/dev/vault.yml --vault-password-file ~/.ansible_vault_pass
# Проверь содержимое authorized_keys
cat ~/.ssh/authorized_keys
# Проверь, есть ли там твой публичный ключ
cat ~/.ssh/authorized_keys | grep "$(cat ~/.ssh/id_rsa.pub)"
# Скопируй ключ:
cd ~/ansible/moex_k8s
ssh-copy-id ilya@192.168.0.44
Введи пароль пользователя ilya на хосте gnld, когда запросит
# Проверь что ключ установлен на удаленном хосте
ssh ilya@192.168.0.44 "echo 'SSH works'"


##### Сети ============
# Проверь сеть:
ansible k8s -m shell \
-a "docker network ls | grep app-net" \
--vault-password-file ~/.ansible_vault_pass
# Проверь сеть
ansible k8s -m shell \
-a "docker inspect tomcat | grep -i 'networkmode'" \
--vault-password-file ~/.ansible_vault_pass
# Проверь доступность PostgreSQL
ansible k8s -m shell \
-a "docker exec tomcat ping -c 2 postgres" \
--vault-password-file ~/.ansible_vault_pass

# Показать все контейнеры с сетями
docker ps --format "table {{.Names}}\t{{.Networks}}"
# Проверь, что сеть mystand-app-net существует:
docker network ls | grep mystand-app-net
# Проверить все сети в Docker:
docker network ls
#  Удалить старую сеть app-net:
docker network rm app-net
# Найди все вхождения app-net в проекте
cd ~/ansible/moex_k8s
grep -r "app-net" --include="*.yml" --include="*.j2" .
# Проверь, какие контейнеры в сети mystand-app-net:
docker network inspect mystand-app-net --format='{{range .Containers}}{{.Name}} {{end}}'
