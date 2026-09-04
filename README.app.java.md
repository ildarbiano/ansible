# 1. Логика Java-приложения (WAR)
/api/db/health	GET	Проверка подключения к БД	Проверочный

# 2. Структура таблиц PostgreSQL (Схема данных)
# Проверь структуру таблицы:
ansible pgs -m \
shell -a "docker exec postgres psql -U ilya-ansible -d dtbase_1 -c '\d first_pastman_req'" \
--vault-password-file ~/.ansible_vault_pass
Мы создадим две таблицы. Эту схему мы применим через миграцию (Liquibase или Flyway), либо 
просто выполним SQL скрипт при первом старте приложения через Spring Boot (spring.jpa.hibernate.ddl-auto=update).
Для простоты пока используем update, чтобы не усложнять на первом этапе.
----- 3. Как и когда мы настроим таблицы?
У нас есть 3 варианта (выбери тот, который тебе ближе):
# Вариант А: 
Инициализация через init-скрипт PostgreSQL (Самый простой)
Мы просто дополним наш файл init-db.sql, который уже лежит на хосте pgs в папке /opt/postgres/init.
При первом запуске контейнера PostgreSQL он выполнит этот скрипт и создаст таблицы.
Плюс: Никакого Java-кода для миграций.
Минус: Если мы захотим менять структуру позже, придется лезть в скрипты вручную.

# Вариант Б: 
Миграции через Java (Spring Boot + Flyway/Liquibase) (Профессионально)
При старте Spring Boot приложение само проверит версию схемы и применит нужные изменения.
Плюс: Это production-стандарт.
Минус: Требует чуть больше кода.

# Вариант С: 
Смешанный (Рекомендую для стенда)
Мы оставим init-db.sql для создания базовых таблиц, а в Java-приложении настроим spring.jpa.hibernate.ddl-auto=validate 
(чтобы оно проверяло, что таблицы есть, но не меняло их само). 
Так мы держим контроль над схемой в руках Ansible, а приложение просто работает с ней.

Настройка пула соединений	HikariCP управляет подключениями

#### C:\app\simple_java_bridge
simple_java_bridge/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── ru/
        │       └── k8s/
        │           └── api/
        │               ├── BackendApplication.java
        │               ├── controller/
        │               │   └── DataController.java
        │               ├── model/
        │               │   └── RequestData.java
        │               ├── repository/
        │               │   └── RequestDataRepository.java
        │               └── service/
        │                   └── DatabaseService.java
        └── resources/
            └── application.properties
mkdir -p src/{main,java,k8s,api}
# Узнай, где ты сейчас:
powershell
get-location
# Запусти создание:
powershell
new-item -itemtype directory -force -path "src\main\java\k8s\api"
new-item -itemtype directory -force -path "src\main\java\k8s\api\controller", "src\main\java\k8s\api\model", "src\main\java\k8s\api\repository","src\main\java\k8s\api\service"
# Проверь, что создалось:
powershell
get-childitem -recurse -directory src
# # В корне проекта (где pom.xml)
mvn clean package
# Архитектура приложения
Пользователь (Postman/Browser)
        │
        ▼
┌─────────────────────────────────────────────────┐
│  Controller (DataController.java)              │
│  - Принимает HTTP-запросы                      │
│  - /api/health, /api/data, /api/data/count     │
│  - Вызывает Service                            │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  Service (DatabaseService.java)                │
│  - Бизнес-логика                               │
│  - Проверка подключения к БД                   │
│  - Сохранение/чтение данных                    │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  Repository (RequestDataRepository.java)       │
│  - JPA репозиторий                             │
│  - Взаимодействие с БД (CRUD)                 │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  Model (RequestData.java)                     │
│  - Entity (таблица requests)                   │
│  - Поля: id, request_time, method, data,      │
│          response_time_ms                      │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│  PostgreSQL (192.168.0.66:5434/dtbase_1)       │
└─────────────────────────────────────────────────┘
#### src/main/resources/application.properties 
лежат настройки подключения к PostgreSQL на хосте pgs (192.168.0.66), приложение будет работать в Tomcat на хосте k8s и обращаться к БД на pgs.
# JPA (Java Persistence API) JPA/Hibernate 
— это стандарт в Java для работы с реляционными базами данных через объекты. Проще говоря, он позволяет не писать SQL-запросы вручную, а оперировать обычными Java‑классами — а JPA сама превратит их в операции с таблицами. spring.jpa.hibernate.ddl-auto=update — говорит Hibernate:
"Сравни мою Java-модель с таблицей в БД. Если таблицы нет — создай. Если есть — добавь новые колонки, но не удаляй старые."
    update	        Создаёт/обновляет таблицы (без удаления)        — для разработки ✅
    create	        Создаёт таблицы при старте (удаляет старые)     — опасно!
    create-drop	    Создаёт при старте, удаляет при остановке       — только для тестов
    validate	    Проверяет, что таблицы соответствуют модели     — для продакшена
    none	        Ничего не делает                                — для продакшена
# Модель RequestData.java 
— описывает структуру таблицы
— это чертёж таблицы requests:
    Какое имя у таблицы
    Какие колонки
    Какого типа каждая колонка
    Какая колонка первичный ключ
аннотация:
@Entity	Класс           это таблица в БД
@Table(name = "...")    Имя таблицы
@Id	Поле                первичный ключ
@GeneratedValue	        Автоинкремент
@Column(name = "...")   Имя колонки в БД
# Controller (DataController) 
— Принимает HTTP-запросы (JSON)
— это точка входа для всех HTTP-запросов
    Слушает входящие запросы
    Понимает, что за запрос пришёл (GET/POST)
    Направляет его в нужный метод
    Возвращает ответ обратно
+Controller првращает	JSON → Java (автоматически через Spring)
Схема потока запроса:
1. Пользователь отправляет запрос
   POST https://192.168.0.55/api/data
   Body: {"id": 1, "value": "test"}

2. Nginx принимает запрос (порт 443)
   → проксирует на Tomcat (порт 8080)

3. Tomcat получает запрос
   → передаёт в Spring Boot приложение

4. Spring Boot ищет контроллер
   → находит DataController с @RequestMapping("/api")
   → находит метод с @PostMapping("/data")

5. Метод createData() выполняется
   → принимает JSON из тела запроса (@RequestBody)
   → вызывает DatabaseService для сохранения в БД

6. Ответ возвращается обратно пользователю
   → JSON с сохранёнными данными

# Service (DatabaseService)
Бизнес-логика, координация
Service выполняет Бизнес-логику (считает время, проверяет доступность)
HTTP-запрос (JSON)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  DataController                                         │
│  - Принимает JSON                                       │
│  - Вызывает DatabaseService                             │
└─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  DatabaseService                                        │
│  - Содержит БИЗНЕС-ЛОГИКУ                               │
│  - Вызывает Repository                                  │
│  - Считает время ответа                                 │
│  - Проверяет доступность БД                             │
└─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  Repository (JPA)                                       │
│  - Превращает Java-объекты в SQL                       │
│  - Выполняет запросы к БД                              │
│  - Возвращает Java-объекты                             │
└─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  PostgreSQL                                             │
└─────────────────────────────────────────────────────────┘
JSON → Java (делает Spring автоматически)
Java → SQL (делает JPA / Hibernate) JPA сам генерирует SQL

# Repository (RequestDataRepository)
— это интерфейс для общения с БД через JPA/Hibernate.  
— Интерфейс JPA/Hibernate, который Превращает Java → SQL и обратно
Repository (JPA)	Java → SQL (автоматически через Hibernate)
JPA/Hibernate делает всю работу за кулисами:
    Java → SQL — превращает вызов save() в INSERT/UPDATE
    SQL → Java — превращает результат SELECT в объект RequestData
    Создание таблиц — по аннотациям в RequestData.java
>>> даёшь Java-объект → JPA сохраняет его в БД
<<< Ты просишь список → JPA достаёт из БД и превращает в Java-объекты
# Запуск playbook
ansible-playbook playbooks/application-deploy.yml \
--vault-password-file ~/.ansible_vault_pass

====================== Deploy в контейнер Tomcat =============================
mkdir files
touch playbooks/application-deploy.yml
ansible-playbook playbooks/application-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение действительно развернулось:
bash
ansible k8s -m shell -a "docker exec tomcat ls -la /usr/local/tomcat/webapps/" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
##### Logs ##############
# Логи. В контейнере Tomcat настроен на ротацию логов через catalina.YYYY-MM-DD.log, а catalina.out отключён (это часто делают в Docker-контейнерах, чтобы логи не забивали диск и управлялись через docker logs).
ansible k8s -m shell -a "docker logs tomcat --tail 20" \
--vault-password-file ~/.ansible_vault_pass
# хвост лога приложения backend
ansible k8s -m shell \
-a "docker logs backend --tail 30" \
--vault-password-file ~/.ansible_vault_pass
# поиск ошибок в полном логи приложения backend
ansible k8s -m shell \
-a "docker logs backend 2>&1 | grep -i 'exception\|error' | tail -30" \
--vault-password-file ~/.ansible_vault_pass
# какие логи Spring Boot есть:
bash
ansible k8s -m shell \
-a "docker exec tomcat ls -la /usr/local/tomcat/logs/" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Логи через docker logs (самый надёжный способ):
bash
ansible k8s -m shell \
-a "docker logs tomcat 2>&1 | tail -50" \
--vault-password-file ~/.ansible_vault_pass
---------
ansible k8s -m shell \
-a "docker logs tomcat 2>&1 | grep -i 'started\|exception\|error' | tail -30" \
--vault-password-file ~/.ansible_vault_pass
# логи Spring Boot в Tomcat:
ansible k8s -m shell \
-a "docker exec tomcat cat /usr/local/tomcat/logs/catalina.out | grep -i 'spring\|started\|error' | tail -30" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Логи на сегодня
ansible k8s -m shell \
-a "docker exec tomcat cat /usr/local/tomcat/logs/catalina.2026-08-27.log | tail -50" \
--vault-password-file ~/.ansible_vault_pass

###### Проверить #################
# Проверь, что Main класс есть:
bash
ansible k8s -m shell \
-a "docker exec tomcat find /usr/local/tomcat/webapps/ROOT/WEB-INF/classes -name '*.class' | grep -i backend" \
--vault-password-file ~/.ansible_vault_pass
#  Проверь, что внутри ROOT есть классы:
ansible k8s -m shell \
-a "docker exec tomcat ls -la /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/" \
--vault-password-file ~/.ansible_vault_pass
-------------
ansible k8s -m shell \
-a "docker exec tomcat find /usr/local/tomcat/webapps/ROOT/WEB-INF/classes -name '*.class' | head -20" \
--vault-password-file ~/.ansible_vault_pass
# Проверь структуру внутри WAR:
bash
ansible k8s -m shell \
-a "docker exec tomcat ls -la /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/k8s/" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что класс BackendApplication есть в нужном месте:
bash
ansible k8s -m shell \
-a "docker exec tomcat ls -la /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/k8s/api/" \
--vault-password-file ~/.ansible_vault_pass
# проверь, что в WAR есть Spring Boot:
bash
ansible k8s -m shell \
-a "docker exec tomcat ls -la /usr/local/tomcat/webapps/ROOT/WEB-INF/lib/ | grep spring" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что Spring Boot запускается через docker logs с более детальным выводом:
ansible k8s -m shell \
-a "docker logs tomcat 2>&1 | grep -i 'exception\|error\|failed'" \
--vault-password-file ~/.ansible_vault_pass


#### порт, port ###########
# Проверь, что приложение слушает на порту 8080:
ansible k8s -m shell \
-a "docker exec tomcat curl -s http://localhost:8080/actuator/health" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение доступно через порт 8080 (не 8081):
# Проверь, что приложение доступно изнутри Tomcat через localhost:
ansible k8s -m shell \
-a "docker exec tomcat wget -q -O- http://localhost:8080/health 2>&1 || echo 'wget failed'" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение слушает порт внутри контейнера:
ansible k8s -m shell \
-a "docker exec tomcat netstat -tlnp 2>/dev/null || ss -tlnp" \
--vault-password-file ~/.ansible_vault_pass

### Tomcat
# Проверь, что Tomcat развернул приложение правильно:
ansible k8s -m shell \
-a "docker exec tomcat cat /usr/local/tomcat/conf/server.xml | grep -i 'port'" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что Spring Boot запустился внутри Tomcat:
ansible k8s -m shell \
-a "docker logs tomcat 2>&1 | grep -i 'started\|application'" \
--vault-password-file ~/.ansible_vault_pass

##### ЗАПУСК jar
# Проверь, что приложение запускается напрямую через JAR:
bash
ansible k8s -m shell \
-a "docker exec tomcat java -jar /usr/local/tomcat/webapps/ROOT.war --server.port=8081 &" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что java -jar действительно запускается с выводом:
bash
ansible k8s -m shell \
-a "docker exec tomcat timeout 10 java -jar /usr/local/tomcat/webapps/ROOT.war --server.port=8081 2>&1 | head -50" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что в application.properties нет ошибок:
bash
ansible k8s -m shell \
-a "docker exec tomcat cat /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application.properties" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение слушает порт:
bash
ansible k8s -m shell \
-a "docker exec tomcat netstat -tlnp | grep -E '8080|8081'" \
--vault-password-file ~/.ansible_vault_pass
------------
ansible k8s -m shell \
-a "docker exec tomcat curl -s http://localhost:8081/health" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение запускается и сразу падает:
bash
ansible k8s -m shell \
-a "docker exec tomcat sh -c 'java -jar /usr/local/tomcat/webapps/ROOT.war --server.port=8081 2>&1 & sleep 5; ps aux | grep java'" \
--vault-password-file ~/.ansible_vault_pass


##### запуск приложения напрямую через JAR (без Tomcat):
ansible k8s -m shell \
-a "docker exec tomcat timeout 15 java -jar /usr/local/tomcat/webapps/ROOT.war 2>&1 | grep -i 'started\|error'" \
--vault-password-file ~/.ansible_vault_pass


##### Проверки к Postgres #############################
# Проверь доступность PostgreSQL из контейнера Tomcat:
bash
# Проверь, что порт 5434 доступен из контейнера
ansible k8s -m shell \
-a "docker exec tomcat nc -zv 192.168.0.66 5434" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что PostgreSQL отвечает
ansible k8s -m shell \
-a "docker exec tomcat curl -s http://192.168.0.66:5434" \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что контейнеры в одной сети:
# Проверь сеть у Tomcat
ansible k8s -m shell \
-a "docker inspect tomcat | grep -i 'networkmode'" \
--vault-password-file ~/.ansible_vault_pass
# Проверь сеть у PostgreSQL
ansible pgs -m shell \
-a "docker inspect postgres | grep -i 'networkmode'" \
--vault-password-file ~/.ansible_vault_pass
# Проверь доступность PostgreSQL
ansible k8s -m shell \
-a "docker exec tomcat ping -c 2 postgres" \
--vault-password-file ~/.ansible_vault_pass
# Проверь доступность PostgreSQL другим способом:
# 1. Проверь через psql клиент (если есть):
ansible k8s -m shell \
-a "docker exec tomcat psql -h postgres -U ilya-ansible -d dtbase_1 -c 'SELECT 1'" \
--vault-password-file ~/.ansible_vault_pass
# 2. Проверь через wget или curl (если есть):
ansible k8s -m shell \
-a "docker exec tomcat curl -s http://postgres:5434" \
--vault-password-file ~/.ansible_vault_pass
# 3. Проверь через nc (если есть):
ansible k8s -m shell -a "docker exec tomcat nc -zv postgres 5432" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# 4. Проверь через getent (есть почти везде):
ansible k8s -m shell \
-a "docker exec tomcat getent hosts postgres" \
--vault-password-file ~/.ansible_vault_pass
# 5. Проверь через nslookup:
ansible k8s -m shell -a "docker exec tomcat nslookup postgres" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение работает через /api/health:
curl -k https://192.168.0.55/api/health


============= Приложение в отдельном контейнере ================================
┌─────────────────────────────────────────────────────────────────┐
│                    Сеть mystand-app-net                         │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  Nginx  │  │  Tomcat  │  │   App    │  │ Postgres │          │
│  │  :443   │  │  :8080   │  │  :8081   │  │  :5432   │          │
│  └─────────┘  └──────────┘  └──────────┘  └──────────┘          │
│                                                                 │
│  Nginx → App (8081)                                             │
└─────────────────────────────────────────────────────────────────┘
# Проверь, что PostgreSQL доступен из контейнера backend:
ansible k8s -m shell -a "docker exec backend ping -c 2 postgres" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
#
ansible k8s -m shell -a "docker ps -a | grep backend" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
#
ansible k8s -m shell -a "docker logs backend --tail 50" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
#
ansible k8s -m shell -a "docker inspect backend | grep -i 'networkmode'" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# логи
ansible k8s -m shell -a "docker logs backend --tail 30" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# posgres psql
ansible pgs -m shell -a "docker exec postgres psql -U ilya-ansible -d dtbase_1 -c '\dt'" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass


#### Postgres =====================
ansible pgs -m shell -a "docker exec postgres psql -U ilya-ansible -d dtbase_1 -c '\dt'" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass
# Проверь, что таблица пустая, но существует
ansible pgs -m shell -a "docker exec postgres psql -U ilya-ansible -d dtbase_1 -c '\d first_pastman_req'" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass

#### 1. Проверь  через Nginx:
bash
curl -k https://192.168.0.55/api/health
Должно вернуть:
json
{"status":"UP","database":"connected"}
# 2. Проверь API через Nginx:
# GET /api/data (список записей)
curl -k https://192.168.0.55/api/data
# 3. Проверь POST через Nginx:
# POST /api/data (создать запись)
curl -k -X POST https://192.168.0.55/api/data \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "value": "test", "timestamp": "2026-08-31T12:00:00"}'
# 4. Проверь, что Nginx проксирует правильно:
# Проверь логи Nginx
ansible k8s -m shell -a "docker logs nginx --tail 10" -i inventories/dev/hosts.yml --vault-password-file ~/.ansible_vault_pass


### MD5 ####
md5sum files/ROOT.war
# PowerShell
PS C:\app\simple_java_bridge> 
Get-FileHash -Path .\target\ROOT.war -Algorithm MD5
# PowerShell маленькими буквами
PS C:\app\simple_java_bridge> 
certutil -hashfile .\target\ROOT.war MD5
# PowerShell <БОЛЬШИМИ> буквами
PS C:\app\simple_java_bridge> 
Get-FileHash .\target\ROOT.war -Algorithm MD5 | Select-Object Hash
# Linux Ansible
ansible k8s -m shell \
-a "md5sum /opt/backend/app.war" \
--vault-password-file ~/.ansible_vault_pass