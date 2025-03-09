# Case-study оптимизации

## Актуальная проблема
В проекте dev.to нашли проблему - долгая загрузка стартовой страницы. Я решила исправить эту проблему
Первый запуск страницы ~ 17 секунд, дальнейшая работа ~ 1.1 секунды

## Часть 1
## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику:
1) Время загрузки стартовой страницы должно укладываться в 0.3 секунды

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался NewRelic

### Ваша находка №1
подала нагрузку через ab и посмотрела на результаты:
```
Server Software:
Server Hostname:        localhost
Server Port:            3000

Document Path:          /?pp=disable
Document Length:        158916 bytes

Concurrency Level:      5
Time taken for tests:   86.244 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      15945400 bytes
HTML transferred:       15891600 bytes
Requests per second:    1.16 [#/sec] (mean)
Time per request:       4312.189 [ms] (mean)
Time per request:       862.438 [ms] (mean, across all concurrent requests)
Transfer rate:          180.55 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   1.0      2       8
Processing:   793 4244 548.1   4147    5170
Waiting:      791 4241 547.9   4138    5168
Total:        801 4246 547.8   4149    5173

Percentage of the requests served within a certain time (ms)
  50%   4149
  66%   4338
  75%   4653
  80%   4750
  90%   4992
  95%   5133
  98%   5162
  99%   5173
 100%   5173 (longest request)
```

Посмотрим на результаты в newRelic.
4.49 s              Average response time
4.37 s              Median response time
5.23 s              95th percentile response time
5.32 s              99th percentile response time
0                   Apdex score
0 %                 Average error rate
6.67 rpm            Average throughput

Один запрос:
```
| Component                                  | Count | Duration | Percentage | Slow spans |
|--------------------------------------------|-------|----------|------------|------------|
| articles/_single_story.html.erb/Partial    | 24    | 1526 ms  | 29.21%     | 0          |
| stories/_main_stories_feed.html.erb/Partial| 1     | 1336 ms  | 25.57%     | 0          |
| stories#index                              | 1     | 934 ms   | 17.88%     | 0          |
| articles/index.html.erb/Rendering          | 1     | 726 ms   | 13.89%     | 0          |
| layouts/application.html.erb/Rendering     | 1     | 277 ms   | 5.3%       | 0          |
| Postgres User find                         | 2     | 130 ms   | 2.49%      | 0          |
| Other                                      | 133   | 296 ms   | 5.67%      | 0          |
```


Попробуем закешировать articles/single_story
Результаты:
```
Server Software:
Server Hostname:        localhost
Server Port:            3000

Document Path:          /?pp=disable
Document Length:        157624 bytes

Concurrency Level:      5
Time taken for tests:   19.581 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      15816200 bytes
HTML transferred:       15762400 bytes
Requests per second:    5.11 [#/sec] (mean)
Time per request:       979.025 [ms] (mean)
Time per request:       195.805 [ms] (mean, across all concurrent requests)
Transfer rate:          788.82 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   1.0      2       7
Processing:   254  958 119.6    963    1261
Waiting:      249  954 119.3    960    1261
Total:        261  960 119.3    966    1262

Percentage of the requests served within a certain time (ms)
  50%    966
  66%    989
  75%   1017
  80%   1033
  90%   1110
  95%   1165
  98%   1203
  99%   1262
 100%   1262 (longest request)
```
New Relic:
1.11 s              Average response time
1.11 s              Median response time
1.3 s               95th percentile response time
1.38 s              99th percentile response time
0.5                 Apdex score
0 %                 Average error rate
33 rpm              Average throughput

Один запрос:
```
| Component                                  | Count | Duration    | Percentage | Slow spans |
|--------------------------------------------|-------|-------------|------------|------------|
| articles/index.html.erb/Rendering          | 1     | 318.06 ms   | 30.29%     | 0          |
| layouts/application.html.erb/Rendering     | 1     | 202.58 ms   | 19.29%     | 0          |
| stories#index                              | 1     | 134.29 ms   | 12.79%     | 0          |
| Postgres Article find                      | 2     | 91.16 ms    | 8.68%      | 0          |
| articles/_sidebar_additional.html.erb/Partial | 1  | 68.5 ms     | 6.52%      | 0          |
| Rack::MiniProfiler#call                    | 1     | 67.9 ms     | 6.47%      | 0          |
| Other                                      | 108   | 167.52 ms   | 15.95%     | 0          |
```

Если поменять параметр cuncurrency level на 1, то результаты будут:
```
Server Software:
Server Hostname:        localhost
Server Port:            3000

Document Path:          /?pp=disable
Document Length:        157624 bytes

Concurrency Level:      1
Time taken for tests:   20.940 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      15816200 bytes
HTML transferred:       15762400 bytes
Requests per second:    4.78 [#/sec] (mean)
Time per request:       209.397 [ms] (mean)
Time per request:       209.397 [ms] (mean, across all concurrent requests)
Transfer rate:          737.62 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   0.8      2       8
Processing:   180  207  25.4    201     330
Waiting:      180  207  25.4    200     330
Total:        182  209  25.4    203     332

Percentage of the requests served within a certain time (ms)
  50%    203
  66%    212
  75%    216
  80%    222
  90%    235
  95%    264
  98%    308
  99%    332
 100%    332 (longest request)
```
NewRelic:
213 ms              Average response time
203 ms              Median response time
290 ms              95th percentile response time
370 ms              99th percentile response time
1                   Apdex score
0 %                 Average error rate
29 rpm              Average throughput

Один запрос:
```
| Component                                   | Count | Duration    | Percentage | Slow spans |
|---------------------------------------------|-------|-------------|------------|------------|
| articles/index.html.erb/Rendering           | 1     | 70.3 ms     | 26.43%     | 0          |
| layouts/application.html.erb/Rendering      | 1     | 63.14 ms    | 23.74%     | 0          |
| Bullet::Rack#call                           | 1     | 34.35 ms    | 12.91%     | 0          |
| stories#index                               | 1     | 32.17 ms    | 12.09%     | 0          |
| stories/_main_stories_feed.html.erb/Partial | 1     | 19.36 ms    | 7.28%      | 0          |
| ActiveRecord::Migration::CheckPending#call  | 1     | 8 ms        | 3.01%      | 0          |
| Other                                       | 109   | 38.68 ms    | 14.54%     | 0          |
```

В целом, без прараллельности запрос укладывается в бюджет, проверим будет ли укладывтаься в бюджет с прараллельностью в local_production.
Однако было бы неплохо разобравться почему так происходит. Мое предположение, что это происходит из-за блокировки обших ресурсов. Например базы данных.

## Часть 2

Сделаем окружение local_production и проверим скорость на нем
```
Server Software:
Server Hostname:        localhost
Server Port:            3000

Document Path:          /?pp=disable
Document Length:        136405 bytes

Concurrency Level:      5
Time taken for tests:   3.745 seconds
Complete requests:      100
Failed requests:        0
Total transferred:      13684800 bytes
HTML transferred:       13640500 bytes
Requests per second:    26.70 [#/sec] (mean)
Time per request:       187.251 [ms] (mean)
Time per request:       37.450 [ms] (mean, across all concurrent requests)
Transfer rate:          3568.48 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   0.7      2       5
Processing:   107  175  48.3    163     336
Waiting:      103  169  48.2    156     336
Total:        109  177  48.1    166     337

Percentage of the requests served within a certain time (ms)
  50%    166
  66%    180
  75%    192
  80%    208
  90%    265
  95%    287
  98%    330
  99%    337
 100%    337 (longest request)
```
NewRelic:
180 ms              Average response time
130 ms              Median response time
254 ms              95th percentile response time
332 ms              99th percentile response time
0.99                Apdex score
0.33 %              Average error rate
10 rpm              Average throughput

## Результаты
В результате проделанной оптимизации удалось ускорить загрузку до ~1 секунды в development окружении.
В local_production время ускорилось до ~0.2 секунд
