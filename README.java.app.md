# 1. Логика Java-приложения (WAR)
/api/db/health	GET	Проверка подключения к БД	Проверочный

# 2. Структура таблиц PostgreSQL (Схема данных)
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


====================== Deploy =============================
mkdir files
touch playbooks/application-deploy.yml
ansible-playbook playbooks/application-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
# Проверь, что приложение действительно развернулось:
bash
ansible k8s -m shell -a "docker exec tomcat ls -la /usr/local/tomcat/webapps/" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Логи
ansible k8s -m shell -a "docker logs tomcat --tail 20" \
--vault-password-file ~/.ansible_vault_pass



