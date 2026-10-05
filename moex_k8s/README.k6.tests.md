Что можно менять в k6:
Параметр	                        Что делает	                                        Влияние
VUs (target)	            - Количество виртуальных пользователей	                    Больше VUs = больше RPS
sleep()	                    - Пауза между итерациями	                                Меньше sleep = больше RPS
duration (в stages)	        - Длительность ступени	                                    Дольше = больше данных
Количество req в итерации	- Сколько запросов за проход	Больше = больше нагрузки
thresholds	                - Пороги для прохождения	Не влияет на нагрузку
        P.S. Есть ещё один параметр — тип сценария:
constant-vus                — постоянное количество
ramping-vus                 — плавное изменение (твой текущий)
constant-arrival-rate       — постоянный RPS
ramping-arrival-rate        — плавный RPS
Но это следующий уровень. Начнём с простых этапов.
=================================================
# Этап 1: Baseline (текущий тест)
Цель: зафиксировать отправную точку.
Параметры:
stages: [
  { duration: '30s', target: 10 },
  { duration: '1m',  target: 50 },
  { duration: '30s', target: 0 },
]
sleep(1)


# Этап 2: Больше VUs
Цель: найти предел приложения по потокам.
Изменение: увеличить VUs в 4 раза.
Параметры:
stages: [
  { duration: '30s', target: 50 },
  { duration: '2m',  target: 200 },  // ← было 50
  { duration: '30s', target: 0 },
]
sleep(1)
Что ищем:
    tomcat_threads_busy             → растёт ли к 200
    hikaricp_connections_pending    → появляется ли
    p95                             → растёт ли выше 100 ms
    Errors                          → появляются ли
Гипотеза: RPS вырастет до ~250-300, потоки будут заняты.


# Этап 3: Меньше sleep
Цель: найти предел по RPS (без изменения VUs).
Изменение: уменьшить sleep до 0.1 сек.
Параметры:
stages: [
  { duration: '30s', target: 50 },
  { duration: '2m',  target: 50 },
  { duration: '30s', target: 0 },
]
sleep(0.1)  // ← было 1
Что ищем:
    RPS                         → растёт ли в 10 раз
    tomcat_threads_busy         → успевает ли обрабатывать
    hikaricp_connections_active → исчерпывается ли пул
    Errors                      → появляются ли
Гипотеза: RPS вырастет до ~500-600, могут появиться ошибки.

# Этап 4: Без sleep (максимум)
Цель: найти абсолютный предел стенда.
Изменение: убрать sleep полностью.
Параметры:
stages: [
  { duration: '30s', target: 50 },
  { duration: '2m',  target: 50 },
  { duration: '30s', target: 0 },
]
// sleep убран
Что ищем:
    RPS                          → какой максимум
    Errors                       → сколько % ошибок
    tomcat_threads_busy          → = 200?
    hikaricp_connections_pending → > 0?
    CPU                          → 100%?
Гипотеза: RPS > 1000, могут появиться 5xx ошибки.


# Этап 5: Стресс-тест (VUs + без sleep)
Цель: найти точку отказа.
Параметры:
stages: [
  { duration: '30s', target: 100 },
  { duration: '2m',  target: 500 },  // ← 500 VUs
  { duration: '30s', target: 0 },
]
// sleep убран
Что ищем:
    Errors       → резко растут
    Latency      → > 1 сек
    Timeout      → появляются
    Стенд упал?
Гипотеза: стенд не выдержит 500 VUs без sleep.

