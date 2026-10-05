# Ключевые концепции, которые нужно знать
В k6 есть несколько базовых сущностей, понимание которых всё упрощает .
  Virtual User (VU):        Это «виртуальный пользователь», который выполняет твой скрипт. Чем больше VU, тем выше нагрузка. Каждый VU независимо выполняет 
  export default function () в цикле:
      text
      VU 1: итерация → sleep → итерация → sleep → итерация ...
      VU 2: итерация → sleep → итерация → sleep → итерация ...
      VU 3: итерация → sleep → итерация → sleep → итерация ...

  Итерация (Iteration):     Один полный проход по скрипту (одно выполнение функции "export default function").  
      javascript
      export default function () {
        const health = http.get(`${baseUrl}/api/health`);   // 1-й запрос.  50 ms
        const post = http.post(`${baseUrl}/api/data`, ...); // 2-й запрос.  50 ms
        const data = http.get(`${baseUrl}/api/data`);       // 3-й запрос.  50 ms
        sleep(1);                                           // пауза.       1000 ms
      }                                                     // Итого:       1150 ms = 1min 150ms. Длительность итерации ≈ 1.05 сек
#  Длительность_итерации = (3 × среднее_время_запроса(http_req_duration: avg=18.2ms)) + sleep(1)
  Длительность_итерации = (3 × 18.2 ms ) + 1'000 ms  = 1,05 sec
  Ступень (Stage):          Этап в профиле нагрузки. Например, «за 30 секунд разогнаться до 50 пользователей». это, этап теста, расписание изменения количества VUs, этап теста с определённым целевым количеством VUs и длительностью.
      javascript
      stages: [
        { duration: '30s', target: 10 },  // Ступень 1
        { duration: '1m',  target: 50 },  // Ступень 2
        { duration: '30s', target: 0 },   // Ступень 3
      ]
      target — это конечная точка ступени. k6 плавно меняет VUs от предыдущего target до текущего target.
  Threshold:                Критерий прохождения/провала теста. Например, «95% запросов должны быть быстрее 500 мс». Если порог нарушен, k6 завершит тест с ошибкой (это полезно для CI/CD).
  Check:                    Мягкая проверка (например, status is 200). Она просто фиксирует успех или неудачу, но не останавливает тест.
# RPS
RPS       = VUs × ( запросов_в_итерации /   длительность_итерации )
# На пике (50 VUs):
  rps     = 50  x (                 3   /   1m 05s                )
  rps     = 50  x (                  2.8571                       ) = 142.85 rps
За 2 минуты:
  Ступень 1 (30 сек):   среднее 5 VUs →   5  × (3/1.05) ≈ 14 rps  * 30  sec = 420 request
  Ступень 2 (1 мин):    среднее 30 VUs →  30 × (3/1.05) ≈ 86 rps  * 60  sec = 5'160 request
  Ступень 3 (30 сек):   среднее 25 VUs →  25 × (3/1.05) ≈ 71 rps  * 30  sec = 2'130 request
  Нельзя складывать RPS ступеней — они идут последовательно, а не параллельно.
                                                                    Итого:  = 7'710 запросов за 2 минуты.
# Средний RPS за тест:
Среднее VUs ≈ (0 + 10 + 50 + 0) /  3-ступени ≈ 20 Vus
  rps       ≈ 20  ×                     2.857                         ≈ 57 rps
Общее_запросов  = Σ   ( Среднее_VUs_ступени × (запросов_в_итерации / длительность_итерации) × длительность_ступени  )
Общее_запросов  =     ( (5 × 2.857 × 30) + (30 × 2.857 × 60) + (25 × 2.857 × 30) )
Нагрузка  = VUs ×   Запросы             x   Итерации
Нагрузка  = 50  x     3                 x     3                     = 300
# Конфигурация к6
######  smoke-тест. Проверка, что стенд живой после деплоя
cat roles/k6/templates/health-check.js.j2
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1,
  duration: '10s',
};
# Только GET /api/health
export default function () {
  const res = http.get('https://{{ hostvars["k8s"]["ansible_host"] }}/api/health'); 
  check(res, {
    'status is 200': (r) => r.status === 200,
    'body contains UP': (r) => r.body.includes('UP'),
  });
  sleep(1);
}
###### Нагрузочный тест
cat roles/k6/templates/load-test.js.j2
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 10 },
    { duration: '1m', target: 50 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const baseUrl = 'https://{{ hostvars["k8s"]["ansible_host"] }}';

  // GET health
  const health = http.get(`${baseUrl}/api/health`);
  check(health, { 'health 200': (r) => r.status === 200 });

  // POST data
  const payload = JSON.stringify({
    user_id: Math.floor(Math.random() * 1000),
    value: `test_${Date.now()}`,
  });
  const post = http.post(`${baseUrl}/api/data`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });
  check(post, { 'post 200': (r) => r.status === 200 });

  // GET data
  const data = http.get(`${baseUrl}/api/data`);
  check(data, { 'data 200': (r) => r.status === 200 });

  sleep(1);
}

# Итоговый Результат
docker logs k6 --tail 50


  █ THRESHOLDS

    http_req_duration
    ✓ 'p(95)<500' p(95)=32.01ms

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%


  █ TOTAL RESULTS

    checks_total.......: 7713    63.900866/s
    checks_succeeded...: 100.00% 7713 out of 7713
    checks_failed......: 0.00%   0 out of 7713

    ✓ health 200
    ✓ post 200
    ✓ data 200

    HTTP
    http_req_duration..............: avg=18.2ms //-----это среднее время ОДНОГО запроса min=6.41ms med=16.63ms max=744.9ms p(90)=27.64ms p(95)=32.01ms
      { expected_response:true }...: avg=18.2ms min=6.41ms med=16.63ms max=744.9ms p(90)=27.64ms p(95)=32.01ms
    http_req_failed................: 0.00%  0 out of 7713
    http_reqs......................: 7713   63.900866/s                 //-------- сколько всего запросов, за время stages (2min)

    EXECUTION
    iteration_duration.............: avg=1.05s  min=1.03s  med=1.05s   max=2.14s   p(90)=1.07s   p(95)=1.08s
    iterations.....................: 2571   21.300289/s
    vus............................: 1      min=1         max=50
    vus_max........................: 50     min=50        max=50

    NETWORK
    data_received..................: 41 MB  337 kB/s
    data_sent......................: 701 kB 5.8 kB/s


