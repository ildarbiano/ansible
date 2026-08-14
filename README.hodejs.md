(Nginx reverse-proxy)
Это профессиональный подход, который используется в реальных проектах.

Схема работы:
text
1. Пользователь открывает https://ваш-сервер/
2. Nginx на порту 443 принимает HTTPS-запрос
3. Nginx расшифровывает трафик и передает в Node.js на порт 3001
4. Node.js работает через HTTP (без SSL)

# Создаем роль nodejs (ваш frontend на Node.js)
mkdir -p ~/ansible/moex_k8s/roles/nodejs/{tasks,templates,vars,handlers,defaults}
# Создаем 
│   ├── nodejs
│   │   ├── defaults
│   │   ├── handlers
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   ├── index.html.j2
│   │   │   ├── package.json.j2
│   │   │   └── server.js.j2
├── playbooks
│   ├── nodejs-deploy.yml


┌─────────────────────────────────────────────────────┐
│                    Хост k8s                         │
│  ┌─────────────────────────────────────────────┐   │
│  │         Docker Network: "app-net"           │   │
│  │                                             │   │
│  │  ┌─────────-┐  ┌──────────┐  ┌──────────┐  │   │
│  │  │  Nginx   │  │  Node.js │  │  Tomcat  │  │   │
│  │  │  :443    │→ │  :3001   │  │  :8080   │  │   │
│  │  └─────────-┘  └──────────┘  └──────────┘ 	|   |
|  | Nginx (80) → Tomcat (8080) 				|   |
|  |            → Node.js (3001) - UI  			 │   │
|  |_____________________________________________|
|____________________________________________________
  
  
-----------------------------------------------------
|                  Хост pgs						    |
|  ______________________________________________
│  │  ┌─────────┐  ┌──────────┐                 │   │
│  │  │Postgres │  │Prometheus│                 │   │
│  │  │  :5434  │  │  :9090   │                 │   │
│  │  └─────────┘  └──────────┘                 │   │
│  └────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────┘

# 1. Проверка синтаксиса
ansible-playbook playbooks/nodejs-deploy.yml \
  --syntax-check \
  -i inventories/dev/hosts.yml \
  --vault-password-file ~/.ansible_vault_pass \
  -v
# Запускаем деплой Node.js
ansible-playbook playbooks/nodejs-deploy.yml --vault-password-file ~/.ansible_vault_pass

=============================================
# 1. Проверяем, запущен ли контейнер
docker ps | grep nodejs

# Проверяем health endpoint
curl http://192.168.0.55:3001/health
# Проверяем метрики
curl http://192.168.0.55:3001/api/metrics
###	проверяем Node.js UI
#	в браузере, напрямую
http://192.168.0.55:3001
# 	через Nginx 
https://192.168.0.55/ui/


====== Редизайн GUI ===========================
#  Создаем структуру файлов
mkdir -p roles/nodejs/templates/tabs
# Создаем пустые файлы для вкладок
touch roles/nodejs/templates/tabs/tis.html roles/nodejs/templates/tabs/ais.html roles/nodejs/templates/tabs/rais.html
# Обновляем roles/nodejs/tasks/main.yml
# Обновляем server.js.j2 с указанием пути
# Обновляем index.html.j2 с указанием пути
# Наполняем пустые файлы для вкладок roles/nodejs/templates/tabs
# 1. Создайте правильную структуру
mkdir -p roles/nodejs/files/public/images
# Скопируйте картинку
cp /path/to/your/image.png roles/nodejs/templates/public/images/
# Добавьте в tasks/main.yml копирование изображений
# Проверяем host
ansible k8s -m shell -a "ls -la /opt/nodejs-app/public"  --vault-password-file ~/.ansible_vault_pass
# Проверяем docker, есть ли директория images docker
ansible k8s -m shell -a "docker exec nodejs-app ls -la /app/public/images/" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# Если есть, проверяем, что в ней лежит
ansible k8s -m shell -a "docker exec nodejs-app ls -la /app/public/" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass