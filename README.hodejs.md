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
│  │  └─────────-┘  └──────────┘  └──────────┘  │   │
|   ---------------------------------------------   |
|----------------------------------------------------
  
  
-----------------------------------------------------
|                  Хост pgs						    |
|  |--------------------------------------------|
│  │  ┌─────────┐  ┌──────────┐                 │   │
│  │  │Postgres │  │Prometheus│                 │   │
│  │  │  :5434  │  │  :9090   │                 │   │
│  │  └─────────┘  └──────────┘                 │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

# 1. Проверка синтаксиса
ansible-playbook playbooks/nodejs-deploy.yml \
  --syntax-check \
  -i inventories/dev/hosts.yml \
  --vault-password-file ~/.ansible_vault_pass \
  -v
# Запускаем деплой Node.js
ansible-playbook playbooks/nodejs-deploy.yml \
  -i inventories/dev/hosts.yml \
  --vault-password-file ~/.ansible_vault_pass

=============================================
# 1. Проверяем, запущен ли контейнер
docker ps | grep nodejs

# 2. Проверяем health endpoint
curl http://192.168.0.55:3001/health

# 3. Проверяем метрики
curl http://192.168.0.55:3001/api/metrics

# 4. Открываем в браузере
http://192.168.0.55:3001