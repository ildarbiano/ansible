# добавить c ньюансом
mkdir host_vars/manager
inventories/dev/hosts.yml
# ssl уже есть
# ping
ansible mngr -m ping \
--vault-password-file ~/.ansible_vault_pass
# Создадим playbook для 
touch playbooks/grafana-deploy.yml
# Создаем роли:
mkdir -p role/docker_setup/{tasks,templates,vars,handlers,defaults}
mkdir -p roles/grafana/{tasks,templates,vars,handlers,defaults}
##### запусти роль docker_setup напрямую: Эта команда напрямую выполняет роль docker_setup на хосте mngr, минуя плейбук. Это полезно для быстрой проверки или разовой задачи
ansible mngr -m include_role -a name=docker_setup \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Запусти docker-setup только для mngr:
ansible-playbook playbooks/docker-setup.yml \
--vault-password-file ~/.ansible_vault_pass \
--limit mngr
# Проверь, что Docker доступен на хосте:
ansible monigen -m shell -a "docker --version" \
--vault-password-file ~/.ansible_vault_pass
# 
ansible-playbook playbooks/grafana-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
#
ansible-playbook playbooks/grafana-deploy.yml \
--vault-password-file ~/.ansible_vault_pass \
--limit mhgr
# 1. Создаем шаблон для provisioning datasource, для подключения к Prometheus удалённо:
cat > roles/grafana/templates/datasource.yml.j2 << 'EOF'
# 2. Создаем шаблон для provisioning dashboard:
bash
cat > roles/grafana/templates/dashboard-provider.yml.j2 << 'EOF'

###### Проверяем
# Проверяем Grafana на Manager
http://192.168.0.33:3000/
# (логин: admin, пароль: admin)
curl http://192.168.0.33:3000
# Проверяем Prometheus на metrics
curl http://192.168.0.34:9100/metrics | head -10
curl http://192.168.0.34:9090/api/v1/query?query=up
http://192.168.0.34:9090
# Проверяем подключение к Prometneus через GUI Grafana:
Connections/Data sources
Explore/prometheus/metrics
# Проверь, что Prometheus слушает на всех интерфейсах:
# На хосте gnld проверь, на каком IP слушает Prometheus
ansible gnld -m shell -a "docker exec prometheus netstat -tlnp | grep 9090" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# Или просто проверь, что порт открыт
ansible gnld -m shell -a "ss -tlnp | grep 9090" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass


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

#### Grafana 
# 1
Всё, что будет сохранено в /var/lib/grafana внутри контейнера, на самом деле будет лежать в папке /opt/grafana/data на хосте monigen (192.168.0.44).
# 2 Второй подход: "Дашборды как код" (более правильный)
Это промышленный подход, который решает вашу задачу по переносу на другую инфраструктуру . Идея в том, чтобы хранить дашборды не в базе данных Grafana, а в виде JSON-файлов вместе с вашим Ansible-кодом. При развертывании Ansible сам "скормит" их Grafana .
Как это будет выглядеть в вашей архитектуре. У вас будет структура в Ansible-проекте:
text
~/ansible/moex_k8s/
├── roles/
│   └── grafana/
│       ├── tasks/
│       │   └── main.yml
│       └── files/                         # <-- Новая папка
│           └── dashboards/                # <-- Для JSON-файлов дашбордов
│               ├── node-exporter-full.json
│               └── postgres-dashboard.json
Вам нужно добавить в roles/grafana/tasks/main.yml блок для копирования папки с JSON-дашбордами и блок для настройки "провайдера" дашбордов 
# Как сохранить созданные дашборды
Пока у вас нет готовых дашбордов, можно:
    1\Экспортировать через интерфейс: Зайти в дашборд → шестеренка → Share → вкладка Export → Save to file .
    2\Положить полученный JSON-файл в папку roles/grafana/files/dashboards/ в вашем Ansible-проекте.
    3\При следующем запуске плейбука этот дашборд появится автоматически.

# Как защитить дашборды от случайной потери:
# Сделай бэкап данных Grafana
ansible monigen -m shell -a "tar -czf /tmp/grafana-backup-$(date +%Y%m%d).tar.gz /opt/grafana/data" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Скопируй бэкап на ansible-master
scp ilya@192.168.0.44:/tmp/grafana-backup-*.tar.gz ~/backups/

#### Пошаговое руководство по импорту
Вот как это делается в вашем интерфейсе Grafana:
1\  В боковом меню наведите курсор на иконку "Дашборды" (четыре квадрата) и выберите "+ Import" .
2\  В поле "Import via grafana.com" введите ID дашборда, например 1860, и нажмите кнопку "Load" .
3\  Система загрузит конфигурацию дашборда. На следующем экране вам нужно будет:
    -Выбрать источник данных (Data Source). Из выпадающего списка выберите ваш источник Prometheus, который вы ранее настроили (например, "Prometheus"). Это ключевой шаг, чтобы дашборд показывал именно ваши данные .
    -При желании, можно изменить имя дашборда и папку для его хранения.
4\ Нажмите синюю кнопку "Import" . Готово, дашборд сразу появится на экране.

📋 Полезные ID дашбордов для вашего стенда. Вот несколько готовых номеров, которые могут пригодиться:
ID дашборда	Что он показывает	Источник данных
1860	Системные метрики: CPU, память, диски, сеть	Node Exporter 
9628	Метрики PostgreSQL: активность, размеры, блокировки	Postgres Exporter
12708	Статистика Nginx: запросы, соединения, статусы	Nginx Exporter
Когда импортируете, не забывайте подставлять свой ID и выбирать правильный источник данных. Если какой-то дашборд не показывает данные, скорее всего, забыли выбрать Prometheus на шаге 3.
### deploy
ansible-playbook playbooks/grafana-deploy.yml --vault-password-file ~/.ansible_vault_pass 
### проверить, что файлы действительно скопировались на хост
ansible mngr -m shell -a "ls -la /opt/grafana/dashboards/" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass

========== Dashboards ===========================
# Spring Boot приложение - "JVM Memory"
Heap память (Использование кучи)	        jvm_memory_used_bytes{area="heap"}	    
Non-Heap память	(Использование метаспейса)  jvm_memory_used_bytes{area="nonheap"}	
Heap память (Максимальный размер кучи)      jvm_memory_max_bytes{area="heap"}	 
# Spring Boot приложение - "Garbage Collection"
Кол-во GC (Young - Молодое поколение)	    jvm_gc_pause_seconds_count{gc=~".*Young.*"}	
Кол-во GC (Old - Старое поколение)	        jvm_gc_pause_seconds_count{gc=~".*Old.*"}	
Длительность GC - Общая длительность 	    jvm_gc_pause_seconds_sum	  
# Spring Boot приложение - "HTTP и RPS"
RPS в секунду       rate(http_server_requests_seconds_count[1m])	
RPS по статусам	    http_server_requests_seconds_count{status=~"2.."}
RPS по методам	    http_server_requests_seconds_count{method=~"GET"}
# Spring Boot приложение - "Latency"
Средняя задержка	        http_server_requests_seconds_sum / http_server_requests_seconds_count	
P95 - 95-й перцентиль   	histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[1m]))	
P99	- 99-й перцентиль       histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[1m]))	
# Spring Boot приложение - "База данных (HikariCP)"
Активные соединения	            hikaricp_connections_active	
Свободные соединения	        hikaricp_connections_idle	
Максимум соединений	            hikaricp_connections_max	
Количество запросов в БД (сек)	rate(spring_data_repository_invocations_seconds_count[1m])	
---------------------------------------------------------------------------------------------
#  Tomcat — "Потоки"
Текущие потоки - Общее количество потоков   Catalina_Connector_currentThreadCount{port="8080"}	
Занятые потоки - Занятые потоки             Catalina_Connector_currentThreadsBusy{port="8080"}	
#  Tomcat — "Соединения и сессии"
Активные сессии	        Catalina_Session_activeSessions
Созданные сессии	    Catalina_Session_sessionCounter
Отклонённые сессии	    Catalina_Session_rejectedSessions
#  Tomcat — "Обработка запросов"
Принятые запросы	    Catalina_Connector_acceptCount{port="8080"}
Таймаут соединения	    Catalina_Connector_connectionTimeout{port="8080"}
------------------------------------------------------------------------
# Nginx : "Общие метрики Nginx"
Текущие активные соединения	                nginx_connections_active	
Скорость запросов. Запросы в секунду (RPS)	rate(nginx_http_requests_total[1m])	
Распределение по статусам HTTP статусы	    nginx_http_requests_total{status=~"2.."}	
# Nginx : "Соединения"
Активные	            nginx_connections_active
Ожидающие	            nginx_connections_waiting
Читающие	            nginx_connections_reading
Пишущие	                nginx_connections_writing
# Nginx : "Трафик"
Входящий трафик (входящий)	    rate(nginx_http_requests_total[1m])
По методам (GET/POST)	        nginx_http_requests_total{method=~"GET"}
------------------------------------------------------------------------
# PostgreSQL — "Соединения"
Активные соединения	                        pg_stat_database_numbackends
Максимальное количество соединений	        pg_settings_max_connections	
Нагрузка на пул - Процент использования	    pg_stat_database_numbackends / pg_settings_max_connections
# PostgreSQL — "Производительность"
Запросы (сек). Чтение данных	rate(pg_stat_database_tup_returned[1m])	
Запись (сек). Вставка записей	rate(pg_stat_database_tup_inserted[1m])	
Обновление записей          	rate(pg_stat_database_tup_updated[1m])	
Удаления. Удаление записей  	rate(pg_stat_database_tup_deleted[1m])	
# PostgreSQL — "Блокировки и кэш"
Кэш хиты	                pg_stat_database_blks_hit	
Чтение с диска	            pg_stat_database_blks_read  
Кэш-хит-рейт. Hit ratio 	pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)	
# PostgreSQL — "Размер базы"
Размер БД "dtbase_1"    pg_database_size_bytes{datname="dtbase_1"}	
Размер таблиц	        pg_table_size_bytes	Размер таблиц

jvm_gc_pause_seconds_sum{application="simple_java_bridge"}
