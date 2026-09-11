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