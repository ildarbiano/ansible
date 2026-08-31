========= Шаг 1: Создаем роль для развертывания Nginx
cd ~/ansible/moex_k8s
mkdir -p roles/nginx/{tasks,templates,handlers,vars}
cat roles/nginx/tasks/main.yml

========= Шаг 2: Создаем playbook для Nginx
playbooks/01-deploy-nginx.yml

======== Шаг 3: Создаем playbook для проверки Nginx
playbooks/02-check-nginx.yml

=============== проверки:
ansible k8s -m ping
# синтаксическая проверка playbook
ansible-playbook -i inventories/dev/hosts.yml playbooks/01-deploy-nginx.yml --syntax-check
# запуск playbook
ansible-playbook -i inventories/dev/hosts.yml playbooks/01-deploy-nginx.yml -v
ansible-playbook playbooks/01-deploy-nginx.yml -v
# запуск playbook с .ansible_vault_pass
ansible-playbook \
playbooks/nginx-deploy.yml \
--vault-password-file ~/.ansible_vault_pass \
  -v
# запуск через оркестратор sity.yaml
ansible-playbook site.yml \
--vault-password-file ~/.ansible_vault_pass

# после успешного запуска, можно проверить:
# Проверка с ansible-master
curl http://192.168.0.55/health
curl http://192.168.0.55/
# в Браузере
http://192.168.0.55/
# Ожидаемый результат:
/health → healthy
/ → HTML страница с "Nginx is running on k8s host!"
появится html страница
============================
# Создаем SSL сертификат на хосте
bash
ansible k8s -m shell -a "mkdir -p /opt/nginx/ssl && openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /opt/nginx/ssl/server.key \
-out /opt/nginx/ssl/server.crt \
-subj '/C=RU/ST=Moscow/L=Moscow/O=Dev/CN=192.168.0.55'" \
--vault-password-file ~/.ansible_vault_pass
# 1. Создаем шаблон index.html.j2
# 2. Добавляем SSL и обновляем tasks/main.yml
# 3. Обновляем шаблон nginx.conf.j2
# 4. Добавляем handler
# 5. Запускаем плейбук
# 6. Проверяем
bash
# HTTP → HTTPS редирект
curl -I http://192.168.0.55
# HTTPS на статику Nginx
curl -k https://192.168.0.55/
# HTTPS на Node.js UI
curl -k https://192.168.0.55/ui/
# HTTPS на Tomcat API (если есть)
curl -k https://192.168.0.55/api/
-----------------------------------
# Проверяем HTTPS:
# на статику Nginx
curl -k https://192.168.0.55/
	https://192.168.0.55/
# Node.js UI через HTTPS
curl -k https://192.168.0.55/ui/
	https://192.168.0.55/ui/
# Tomcat через HTTP https://192.168.0.55/S (корень)
curl -k https://192.168.0.55/api/
	https://192.168.0.55/api/
# health nginx через HTTPS
curl -k https://192.168.0.55/health
	https://192.168.0.55/health

###### Проверь статус Nginx: ############################
ansible k8s -m shell -a "docker ps -a | grep nginx" \
--vault-password-file ~/.ansible_vault_pass
# Проверь логи Nginx:
ansible k8s -m shell -a "docker logs nginx --tail 20" \
--vault-password-file ~/.ansible_vault_pass