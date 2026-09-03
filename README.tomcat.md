План действий
Шаг	Что делаем	Где
1	Создаем роль tomcat	roles/tomcat/tasks/main.yml
2	Создаем playbook tomcat-deploy.yml	playbooks/tomcat-deploy.yml
3	Запускаем развертывание Tomcat	ansible-playbook playbooks/tomcat-deploy.yml ...
4	Проверяем Tomcat	docker logs tomcat, curl http://k8s:8080
5	Настраиваем Nginx как Reverse Proxy для Tomcat	Обновляем nginx.conf.j2 и перезапускаем Nginx
6	Проверяем полный тракт (Nginx → Tomcat)	curl http://k8s/api/
7	Проверяем связанность Tomcat → PostgreSQL (по логам)	docker logs tomcat на наличие подключения к БД

# 
mkdir -p roles/tomcat/{tasks,templates,handlers,vars}
#
roles/tomcat/tasks/main.yml
bash
cat > roles/tomcat/tasks/main.yml << 'EOF'
---
EOF
# Шаг 2: Создаем handlers для Tomcat
bash
cat > roles/tomcat/handlers/main.yml << 'EOF'
---
# Шаг 3: Создаем playbook tomcat-deploy.yml
bash
cat > playbooks/tomcat-deploy.yml << 'EOF'
---
# Шаг 4: Запускаем Tomcat
ansible-playbook playbooks/tomcat-deploy.yml \
--vault-password-file ~/.ansible_vault_pass -v

# Шаг 5: Проверяем Tomcat
bash
# Проверка с ansible-master
curl http://192.168.0.55:8080

# Логи Tomcat
# Сначала зайдите на хост k8s через SSH
ssh ilya@192.168.0.55

# И уже там выполните:
sudo docker logs tomcat | tail -20
[sudo] password for ilya:

# Или используйте ansible с флагом vault:
bash
ansible k8s -m shell \
-a "docker logs tomcat | tail -20" \
	--vault-password-file ~/.ansible_vault_pass

===== Проверка связанности tomcat-PostgreSQL:
# 1. Проверяем, что контейнер Tomcat вообще видит хост pgs (через DNS или IP)
bash
ansible k8s -m shell \
-a "docker exec tomcat cat /etc/hosts" \
--vault-password-file ~/.ansible_vault_pass
# 2. Проверяем, что порт 5434 доступен из контейнера (через bash и curl)
bash
ansible k8s -m shell -a "docker exec tomcat bash -c 'timeout 3 bash -c \
	"echo > /dev/tcp/192.168.0.66/5434\" && echo \
	"Port 5434 is reachable\
	" || echo \
	"Port 5434 is NOT reachable\"'" \
	--vault-password-file ~/.ansible_vault_pass
# 4. Проверяем доступность хоста pgs с самого хоста k8s
bash
ansible k8s -m shell \
-a "ping -c 2 192.168.0.66" \
--vault-password-file ~/.ansible_vault_pass
	
=====	Настраиваем Nginx как Reverse Proxy для Tomcat
# Обновим конфигурацию Nginx, добавив проксирование на Tomcat:
cat > roles/nginx/templates/nginx.conf.j2 << 'EOF'
EOF
# Перезапускаем Nginx
bash
ansible k8s -m shell -a "docker restart nginx" --vault-password-file ~/.ansible_vault_pass
# 
mcedit roles/nginx/handlers/main.yml
# Запускаем деплой Nginx (он обновит конфигурацию)
ansible-playbook playbooks/nginx-deploy.yml -i \
 --vault-password-file ~/.ansible_vault_pass
 
 
### Проверяем доступ к Tomcat через Nginx (корневой путь)
#	Проверяем API Tomcat через Nginx
https://192.168.0.55/api/
# Проверяем напрямую Tomcat (минуя Nginx) корень Tomcat
curl http://192.168.0.55:8080/


# Проверяем, какие приложения развернуты
ansible k8s -m shell -a "ls -la /opt/tomcat/webapps/"  --vault-password-file ~/.ansible_vault_pass
# Проверяем логи Tomcat
ansible k8s -m shell -a "docker logs tomcat --tail 20" --vault-password-file ~/.ansible_vault_pass