========= Шаг 1: Создаем роль для развертывания Nginx
cd ~/ansible/moex_k8s
mkdir -p roles/nginx/{tasks,templates,handlers,vars}
mkdir -p roles/mtail/tasks
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
curl -k https://192.168.0.55/api/health
curl -k https://192.168.0.55/api/data?page=0&size=10
-----------------------------------
# Войти внутрь контейнера
docker exec -it postgres bash

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


# Мониторинг
# проверка, есть ли метрика
curl -s "http://192.168.0.34:9090/api/v1/label/__name__/values" | tr ',' '\n' | grep http_server_requests


##### НАСТРОКИ NGINX :
Параметры Nginx: значения по умолчанию и что можно менять
Вот основные директивы, которые влияют на производительность и которые стоит знать.

# Worker Processes и Connections
Это фундамент, определяющий, сколько соединений Nginx может обработать.
Директива				Значение по умолчанию				Рекомендация для тестов
worker_processes		1									auto (по одному на ядро CPU) 
worker_connections		512 (в старых версиях) 				1024 или выше, в зависимости от памяти 
worker_rlimit_nofile	Не задан (наследуется от ОС)		2 * worker_connections (например, 2048 при 1024) Важно: Nginx — это прокси. Каждое соединение с клиентом и соединение с backend (Tomcat/Node.js) — это два разных файловых дескриптора (сокета). Поэтому worker_rlimit_nofile должен быть как минимум в 2 раза больше worker_connections, чтобы избежать ошибок «Too many open files» .

# Управление Keep-Alive (Долгие соединения)
Это напрямую влияет на то, как долго висят "Waiting" соединения в stub_status.

Директива				Значение по умолчанию				Рекомендация для тестов
keepalive_timeout		75s (с версии 1.19) 				65s 
keepalive_requests		1000 (с версии 1.19) 				100 или 200. Совет: Уменьши keepalive_requests (например, до 100), чтобы соединения быстрее переиспользовались или закрывались. Это даст более предсказуемую картину в nginx_connections_waiting под нагрузкой.

# Оптимизация ввода-вывода (I/O) и Сети
Эти директивы не про лимиты, а про эффективность передачи данных.

Директива				Значение по умолчанию				Рекомендация
sendfile				off 								on (позволяет отдавать статику напрямую из ядра) 
tcp_nopush				off 								on (эффективнее для больших файлов) 
tcp_nodelay				on 									on (снижает задержку для маленьких пакетов) 

# Кэш открытых файлов (Open File Cache)
Если Nginx часто отдает статику (а твоя /ui/ — это Node.js, но / — статика), этот кэш снижает нагрузку на диск.

Директива					Значение по умолчанию			Рекомендация
open_file_cache	off 		(не кэширует) 					max=1000 inactive=60s 
open_file_cache_valid		60s								80s (как в примерах) 
open_file_cache_min_uses	1								1 (достаточно) 
open_file_cache_errors		off 							on (кэшировать 404) 

# Итоговая стратегия для твоего теста
Учитывая, что тебе нужна предсказуемость для наблюдения за Tomcat:
	worker_connections 		— оставь стандартные 512 или подними до 1024. Ты вряд ли упрешься в лимит на 50 соединениях.
	keepalive_requests 		— снизь до 100. Это заставит Nginx чаще пересоздавать соединения с backend, и ты увидишь более динамичную картину в метриках connections.
	worker_rlimit_nofile 	— добавь директиву явно (1024 при worker_connections 512), чтобы исключить ошибки на уровне ОС.
Учитывай, что эти директивы (кроме server) должны быть в глобальном контексте http, а не внутри server. Твой текущий шаблон монтируется как conf.d/default.conf, поэтому тебе понадобится отдельный глобальный конфиг или монтирование основного nginx.conf.

# Docker
sudo docker exec -it nginx sh


#### Конфигурация nginx:



#### Мониторинг nginx:
Метрика							Значение	Что показывает
request_time=0.023				23 ms		Общее время Nginx (от клиента до ответа)
upstream_response_time=0.024	24 ms		Время backend (Spring Boot)
upstream_connect_time=0.002		2 ms		Время соединения с backend
upstream_header_time=0.023		23 ms		Время до первого байта от backend
Разница:
	request_time (23 ms) 	≈ 	upstream_response_time (24 ms) 			→ Nginx не тормозит
	request_time 			>> 	upstream_response_time 					→ Nginx тормозит
# ntail парсер
curl -s http://192.168.0.55:3903/metrics | grep nginx
# Browser
http://192.168.0.55:3903/
На этой странице должен быть раздел Program Loader, где будет указана конкретная ошибка компиляции программы.
# Проверь ошибки загрузки:
curl -s http://192.168.0.55:3903/metrics | grep mtail_prog_load_errors
curl -s http://192.168.0.55:3903/metrics | grep -E "mtail_prog_load|nginx_lines"
Ожидаем: 0
# Проверь метрики Nginx:
# Запрос
curl -k https://192.168.0.55/api/health
# Метрики
curl -s http://192.168.0.55:3903/metrics | grep nginx
#  связь Nginx ↔ Application:
Метрика									Источник						Что показывает
nginx_request_time_seconds				mtail							Nginx (общее)
nginx_upstream_response_time_seconds	mtail							Backend (через Nginx)
http_server_requests_seconds			Actuator						Backend (напрямую)