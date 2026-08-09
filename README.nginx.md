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
ansible-playbook site.yml --vault-password-file ~/.ansible_vault_pass

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

