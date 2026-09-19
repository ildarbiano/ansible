# Официальный сайт: 
grafana.com/docs/k6
#
mkdir -p roles/k6/{tasks,templates,files,handlers}
touch roles/k6/tasks/main.yml
touch roles/k6/handlers/main.yml
touch roles/k6/templates/health-check.js.j2
touch roles/k6/templates/load-test.js.j2
touch playbooks/k6-deploy.yml
# Запуск:
ansible-playbook playbooks/k6-deploy.yml \
--vault-password-file ~/.ansible_vault_pass
# Просмотр результатов:
# Логи k6
ansible mngr -m shell \
-a "docker logs k6 --tail 50" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass
# Следить в реальном времени
ansible mngr -m shell \
-a "docker logs k6 -f" \
-i inventories/dev/hosts.yml \
--vault-password-file ~/.ansible_vault_pass

#### Важно: k6 запускается один раз, при запущеном стенде, и завершается после теста. Для повторного запуска можно:
# 1/Перезапустить контейнер:
ansible mngr -m shell \
-a "docker restart k6" \
--vault-password-file ~/.ansible_vault_pass
# 2/передеплоить/ Запуск.

# Ключевые концепции, которые нужно знать
В k6 есть несколько базовых сущностей, понимание которых всё упрощает .
  Virtual User (VU):        Это «виртуальный пользователь», который выполняет твой скрипт. Чем больше VU, тем выше нагрузка.
  Iteration:                Один полный проход по твоему скрипту (одно выполнение функции default).
  Stage:                    Этап в профиле нагрузки. Например, «за 30 секунд разогнаться до 50 пользователей».
  Threshold:                Критерий прохождения/провала теста. Например, «95% запросов должны быть быстрее 500 мс». Если порог нарушен, k6 завершит тест с ошибкой (это полезно для CI/CD).
  Check:                    Мягкая проверка (например, status is 200). Она просто фиксирует успех или неудачу, но не останавливает тест.

###### Настройка мониторинга k6
# Метрики k6 в Prometheus. Проверь, что Prometheus принимает Remote Write
curl -s "http://192.168.0.34:9090/api/v1/query?query=k6_http_reqs_total" | head -50
# Если ответ пустой ("result":[]) — метрики ещё не пришли (подожди 15-30 секунд).
###### Проверь отдельные метрики k6
# Количество виртуальных пользователей
curl -s "http://192.168.0.34:9090/api/v1/query?query=k6_vus" | head -30
# RPS (запросов в секунду)
curl -s "http://192.168.0.34:9090/api/v1/query?query=rate(k6_http_reqs_total[1m])" | head -30
# Время ответа p95
curl -s "http://192.168.0.34:9090/api/v1/query?query=k6_http_req_duration_p95" | head -30
# Ошибки
curl -s "http://192.168.0.34:9090/api/v1/query?query=k6_http_req_failed_total" | head -30
# k6 не должен быть в /targets. Его метрики уже доступны через API. Список всех метрик k6:
curl -s "http://192.168.0.34:9090/api/v1/label/__name__/values" | tr ',' '\n' | grep k6
k6_checks_rate	                Процент успешных проверок
k6_data_received_total	        Получено данных (байт)
k6_data_sent_total	            Отправлено данных (байт)
k6_http_req_blocked_seconds	    Время блокировки
k6_http_req_connecting_seconds	Время установки соединения
k6_http_req_duration_seconds	  Время ответа (гистограмма)
k6_http_req_failed_rate	        Процент ошибок
k6_http_req_receiving_seconds	  Время получения
k6_http_req_sending_seconds	    Время отправки
k6_http_req_tls_handshaking_seconds	    TLS handshake
k6_http_req_waiting_seconds	    Время ожидания
k6_http_reqs_total	            Всего запросов
k6_iteration_duration_seconds	  Время итерации
k6_iterations_total	            Всего итераций
k6_vus	                        Виртуальных пользователей сейчас
k6_vus_max	                    Максимум VUs