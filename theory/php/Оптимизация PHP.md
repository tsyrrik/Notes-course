### 0) Диагностика прежде всего

- **HTTP нагрузка**: `wrk`, `k6`, `ab` для синтетики; метрики Nginx/Ingress.
    
- **Профилировщики**:
    
    - **Xdebug profiler** (cachegrind) для локальной детальной трассировки.
        
    - **Blackfire / Tideways** — низкий оверхед, профилирование продакшна.
        
    - **XHProf/uftrace** — бюджетный вариант callgraph.
        
- **Лог медленных**: PHP-FPM slowlog (ниже).
    
- **БД**: slow query log, `EXPLAIN`, трассировка драйвера (PDO).
    
- **Системные**: `strace -c -p <pid>`, `perf top`, iostat/vmstat.
    

---

## 1) PHP-FPM: slowlog и базовая настройка

**Включить slowlog пула**:

```ini
; /etc/php/8.3/fpm/pool.d/www.conf
request_slowlog_timeout = 2s
slowlog = /var/log/php-fpm/www-slow.log

; полезные статусы
pm.status_path = /fpm-status
ping.path = /ping
```

**Подбор процессов**:

```ini
pm = dynamic              ; или static/ondemand
pm.max_children = <RAM на PHP> / <RAM на один воркер>
pm.max_requests = 500     ; защита от утечек
```

Оценить RAM на воркер: посмотри RSS у pids под нагрузкой.

---

## 2) OPcache и JIT

**OPcache** уменьшает парсинг/компиляцию и ускоряет автолоад.  
**JIT** полезен редко (CPU-тяжелые math/regex), для веб-IO обычно бесполезен.

```ini
; opcache
opcache.enable=1
opcache.enable_cli=0
opcache.validate_timestamps=0     ; в проде через деплой-«тычок»
opcache.revalidate_freq=0
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=50000
opcache.save_comments=1
opcache.enable_file_override=1
realpath_cache_size=4096k
realpath_cache_ttl=600

; JIT (экспериментируй осознанно)
opcache.jit_buffer_size=64M
opcache.jit=1205
```

**Preloading** (если уместно):

```ini
opcache.preload=/var/www/app/preload.php
opcache.preload_user=www-data
```

---

## 3) Composer: автолоад и зачистка

**Оптимизировать classmap**:

```bash
composer dump-autoload -o --classmap-authoritative
composer install --no-dev --prefer-dist --no-progress --no-ansi
```

- `--classmap-authoritative` убирает поиск по PSR-4 в рантайме.
    
- Убедись, что весь прод-код действительно попал в classmap.
    

---

## 4) Отключить Xdebug в проде

Xdebug — профилировщик и дебаггер, а не «часть PHP». В проде выключить:

```ini
; /etc/php/8.3/mods-available/xdebug.ini
zend_extension=           ; закомментируй
xdebug.mode=off
```

В Docker делай отдельные образы: dev со Xdebug и prod без.

---

## 5) Сетевые вызовы: параллелить

### `curl_multi` (ядро PHP)

```php
$mh = curl_multi_init();
$chs = [];
foreach ($urls as $u) {
  $ch = curl_init($u);
  curl_setopt_array($ch, [CURLOPT_RETURNTRANSFER=>true, CURLOPT_TIMEOUT=>3]);
  curl_multi_add_handle($mh, $ch);
  $chs[] = $ch;
}
do { $status = curl_multi_exec($mh, $running); curl_multi_select($mh); } while ($running);
$results = array_map(fn($ch)=>curl_multi_getcontent($ch), $chs);
foreach ($chs as $ch) curl_multi_remove_handle($mh, $ch);
curl_multi_close($mh);
```

### Guzzle Promises/Pool

```php
$client = new \GuzzleHttp\Client(['timeout'=>3]);
$promises = array_map(fn($u)=>$client->getAsync($u), $urls);
$results  = \GuzzleHttp\Promise\Utils::unwrap($promises);
```

### Async фреймворки

- **Amp/ReactPHP** — неблокирующий IO, удобны для fan-out запросов.
    

---

## 6) Код и алгоритмы

- Убери N+1 к БД, используй `IN (...)`, JOIN, батчи.
    
- Кэшируй тяжёлые вычисления (Redis, локальный статик-кеш per-request).
    
- Менее аллокаций: избегай лишних копий массивов, `array_merge` в циклах.
    
- JSON: используешь `JSON_THROW_ON_ERROR` и `json_encode(..., JSON_INVALID_UTF8_SUBSTITUTE)`.
    
- Регулярки: компилируй и переиспользуй шаблоны.
    
- Валидации/нормализации — батчи вместо поключевой обработки.
    

---

## 7) БД и внешние ресурсы (в двух словах)

- **Connection pooling**: PgBouncer/MySQL ProxySQL.
    
- Prepared statements, индексы по фактическим фильтрам/сортировкам.
    
- Уменьшить чаты с БД: агрегаты в SQL, не в PHP-цикле.
    
- Для очередей/кэша — Redis, а не БД.
    
- Выноси тяжёлую агрегацию в ClickHouse/материализованные представления (Postgres).
    

---

## 8) Кеширование

- HTTP-кеш: `ETag`, `Cache-Control`, CDN/Reverse-proxy (Varnish/Cloudflare).
    
- Приложение: Redis с тэгами/TTL, прогрев кеша (prefetch).
    
- OPCache уже выше.
    
- В шаблонах: частичное кеширование (фрагменты).
    

---

## 9) PHP-FPM и окружение

- `pm.max_children`: считай по памяти, не «на глаз».
    
- `pm.process_idle_timeout` для ondemand.
    
- Keep-alive, HTTP/2, gzip/brotli — на уровне веб-сервера.
    
- Логи и метрики: Prometheus + php-fpm exporter, статус-страницы.
    

---

## 10) Альтернативные рантаймы

### RoadRunner / Swoole / FrankenPHP

- **Плюсы**: нет форка-на-запрос, DI-контейнер/автолоад тёплые, меньше syscalls, выше RPS.
    
- **Минусы**: долгоживущий процесс → риск утечек и «state bleed». Нужна дисциплина.
    

**Правила миграции**:

- Только **иммутабельные** сервисы и DTO; никаких глобальных синглтонов с mutable-стейтом.
    
- Сброс per-request данных вручную (контейнер-scope, resettable services).
    
- Очистка статических кешей и реинициализация провайдеров на каждый запрос.
    
- Отладить «горячий перезапуск» и graceful shutdown.
    
- Для Symfony/Laravel есть готовые мосты (Symfony Runtime, Laravel Octane/RoadRunner).
    

---

## 11) Быстрые победы (checklist)

-  Выключить Xdebug в проде.
    
-  Включить OPcache и подобрать лимиты.
    
-  `composer dump-autoload -o --classmap-authoritative` и `--no-dev`.
    
-  Настроить PHP-FPM slowlog + разобрать топ-стэктрейсы.
    
-  Параллелить внешние HTTP/SDK вызовы.
    
-  Починить N+1 и добавить нужные индексы.
    
-  Добавить кэш слоёв (HTTP/Redis) и прогрев популярных ключей.
    
-  Вынести тяжёлую агрегацию из PHP в SQL/матвью/ClickHouse.
    
-  Если упёрлись в FPM — пилот RoadRunner/Swoole с дисциплиной стейта.
    

---

## 12) Когда JIT и «альтернативный рантайм» действительно окупятся

- JIT — когда код CPU-bound (линейная алгебра, крипта, парсинг), а не IO-bound.
    
- RoadRunner/Swoole — когда львиная доля CPU уходит на бутстрап/DI/автолоад, а приложение «болтает» с кучей API и БД.
    

---

### Итог

Сначала **померяй**. Потом убери очевидный мусор: Xdebug, неоптимальный автолоад, N+1, отсутствие кеша. Дальше — OPcache тюнинг, параллельные IO, и только затем играйся с JIT и альтернативными рантаймами.

https://www.youtube.com/watch?v=iUDauNxyeUI

[[Продуманная оптимизация]]

[[⚙️ Composer Autoload и оптимизация]]
