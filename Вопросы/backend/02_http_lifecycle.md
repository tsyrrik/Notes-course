## Вопрос: Life cycle HTTP-запроса

## Простой ответ
- Запрос проходит: браузер → DNS/TCP/TLS → веб-сервер → приложение → ответ обратно.
- Веб-сервер лишь проксирует динамику (PHP-FPM и т.п.), статикой отвечает сам.

1. Браузер формирует запрос, DNS резолвит домен, устанавливается TCP/IP, отправляется HTTP.
2. Веб-сервер (Nginx/Apache) принимает и отдаёт в PHP-интерпретатор; код обращается к БД/сервисам, формирует ответ.
3. Ответ возвращается браузеру, который рендерит страницу.

```text
User -> https://site.com/profile
DNS -> IP -> TCP+TLS -> GET /profile
Nginx -> php-fpm -> PHP код -> БД
Ответ HTML -> Браузер рисует страницу
```

## Ответ

### 1. Ввод URL в браузере и формирование запроса

Когда пользователь вводит `https://site.com/profile` в адресную строку, браузер выполняет несколько подготовительных шагов до отправки запроса. Сначала он проверяет, не находится ли ресурс в **локальном кэше** (memory cache, disk cache, Service Worker cache). Если кэш невалиден или отсутствует, браузер разбирает URL на компоненты: схема (`https`), хост (`site.com`), путь (`/profile`), query string и fragment.

### 2. DNS-резолвинг

Браузеру необходимо преобразовать доменное имя в IP-адрес. DNS-резолвинг проходит через цепочку кэшей:

```text
Браузерный DNS-кэш
    ↓ (miss)
OS DNS-кэш (/etc/hosts, systemd-resolved)
    ↓ (miss)
Роутер / провайдерский DNS
    ↓ (miss)
Рекурсивный DNS-резолвер (8.8.8.8, 1.1.1.1)
    ↓
Root DNS → TLD (.com) → Authoritative NS (site.com)
    ↓
Ответ: site.com → 93.184.216.34
```

DNS-запрос обычно идёт по UDP (порт 53), но может переключиться на TCP для ответов >512 байт. Современные браузеры поддерживают DNS-over-HTTPS (DoH) и DNS-over-TLS (DoT) для шифрования DNS-трафика. Результат кэшируется на время, указанное в TTL записи.

### 3. TCP-соединение (three-way handshake)

Получив IP-адрес, браузер устанавливает TCP-соединение:

```text
Клиент                         Сервер
   |──── SYN (seq=x) ──────────→|     1. Клиент инициирует соединение
   |←─── SYN-ACK (seq=y,ack=x+1)|     2. Сервер подтверждает и отвечает
   |──── ACK (ack=y+1) ─────────→|     3. Клиент подтверждает
   |                              |
   |     Соединение установлено   |
```

TCP гарантирует доставку, порядок пакетов и контроль потока. На этом этапе определяется MSS (Maximum Segment Size) и устанавливается TCP Window Size. Для HTTP/2 и HTTP/3 могут использоваться дополнительные оптимизации — TCP Fast Open и 0-RTT.

### 4. TLS Handshake (для HTTPS)

Для HTTPS поверх TCP устанавливается TLS-соединение. В TLS 1.3 (современный стандарт) handshake упрощён до 1-RTT:

```text
Клиент                                Сервер
   |── ClientHello ─────────────────→|  (поддерживаемые cipher suites, ключи)
   |                                  |
   |←── ServerHello + Certificate ───|  (выбранный cipher, сертификат, ключ)
   |←── Finished ────────────────────|
   |                                  |
   |── Finished ─────────────────────→|
   |                                  |
   |   Шифрованный канал установлен   |
```

В процессе TLS handshake:
- Клиент и сервер согласовывают версию протокола и cipher suite
- Сервер предъявляет X.509-сертификат, подписанный CA
- Клиент проверяет цепочку сертификатов (Certificate Chain)
- Стороны обмениваются ключами (ECDHE / X25519) для Perfect Forward Secrecy
- Устанавливается симметричный ключ шифрования (AES-256-GCM, ChaCha20)

### 5. Отправка HTTP-запроса

После установки соединения браузер отправляет HTTP-запрос:

```http
GET /profile HTTP/1.1
Host: site.com
User-Agent: Mozilla/5.0 (Macintosh; ...)
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, br
Accept-Language: ru-RU,en;q=0.9
Cookie: session_id=abc123; csrf_token=xyz
Connection: keep-alive
```

В HTTP/2 запрос отправляется в бинарном формате с использованием HPACK-сжатия заголовков. HTTP/2 позволяет мультиплексировать несколько запросов по одному TCP-соединению, устраняя проблему Head-of-Line Blocking на уровне HTTP.

### 6. Обработка веб-сервером (Nginx)

Nginx принимает соединение через `epoll` (Linux) / `kqueue` (macOS) — событийную модель без блокирующих потоков. Конфигурация определяет, что делать с запросом:

```nginx
server {
    listen 443 ssl http2;
    server_name site.com;

    # Статика — отдаёт Nginx напрямую
    location /static/ {
        root /var/www/site;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Динамика — передаёт в PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Фронт-контроллер (Laravel, Symfony)
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

Для статики (CSS, JS, изображения) Nginx отдаёт файл напрямую через `sendfile()` — системный вызов, который копирует данные из файла в сокет без промежуточного буфера в user-space. Для динамических запросов Nginx передаёт их в backend (PHP-FPM) через FastCGI-протокол.

### 7. Обработка PHP-FPM и выполнение кода

PHP-FPM master-процесс принимает запрос и передаёт его свободному воркеру. Воркер:

```text
1. Инициализирует SAPI (Server API) окружение
2. Заполняет суперглобальные: $_GET, $_POST, $_SERVER, $_COOKIE
3. Загружает index.php (фронт-контроллер фреймворка)
4. Фреймворк: bootstrap → routing → middleware → controller
5. Controller обращается к БД, внешним API, кэшу
6. Формирует HTTP-ответ (HTML/JSON)
7. Возвращает ответ через FastCGI → Nginx
```

### 8. Взаимодействие с базой данных

PHP-код обращается к базе данных (MySQL, PostgreSQL) через persistent connection или connection pool:

```text
PHP-FPM Worker → TCP/Unix Socket → MySQL/PostgreSQL
    |── SQL-запрос (SELECT * FROM users WHERE id = 42)
    |← Результат (строки данных)
    |── Закрытие/возврат соединения в пул
```

### 9. Формирование и отправка HTTP-ответа

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 4521
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
Cache-Control: no-cache, private
X-Request-Id: f47ac10b-58cc-4372-a567-0e02b2c3d479

<!DOCTYPE html>
<html>
<head><title>Профиль</title></head>
<body>...</body>
</html>
```

Nginx может дополнительно сжать ответ через gzip/brotli, добавить security-заголовки и записать строку в access log.

### 10. Рендеринг в браузере

Получив HTML, браузер начинает рендеринг:

```text
HTML-парсинг → DOM Tree
                  ↘
                   → Render Tree → Layout → Paint → Composite
                  ↗
CSS-парсинг  → CSSOM

JavaScript → может модифицировать DOM/CSSOM
```

Параллельно браузер обнаруживает внешние ресурсы (CSS, JS, img, fonts) и отправляет дополнительные HTTP-запросы. Благодаря HTTP/2 мультиплексированию и `keep-alive` многие ресурсы загружаются по уже установленному соединению.

### Полная диаграмма жизненного цикла

```text
┌─────────┐   DNS    ┌─────┐   TCP+TLS   ┌───────┐  FastCGI  ┌─────────┐
│ Браузер │────────→│ DNS │────────────→│ Nginx │─────────→│ PHP-FPM │
│         │         └─────┘             │       │          │ Worker  │
│         │←───────────────────────────│       │←─────────│         │
│         │      HTTP Response          │       │          │    ↓    │
│         │                             └───────┘          │  MySQL  │
│ Render  │                                                └─────────┘
│ DOM+CSS │
│ Execute │
│   JS    │
└─────────┘

Временные затраты (типичный запрос):
DNS lookup:     1–50 ms  (с кэшем ~0 ms)
TCP handshake:  10–50 ms (1 RTT)
TLS handshake:  10–50 ms (1 RTT для TLS 1.3)
TTFB (server):  50–500 ms
Content transfer: 10–200 ms
Render:         50–500 ms
───────────────────────────
Total:          ~130–1350 ms
```

### Оптимизации на каждом этапе

| Этап | Оптимизация |
|------|-------------|
| DNS | DNS prefetch (`<link rel="dns-prefetch">`), низкий TTL для failover |
| TCP | `keep-alive`, TCP Fast Open, HTTP/2 мультиплексирование |
| TLS | TLS 1.3 (1-RTT), OCSP stapling, session tickets |
| Сервер | Nginx `sendfile`, `gzip_static`, `proxy_cache` |
| PHP | OpCache, JIT, preloading, persistent connections |
| БД | Индексы, query cache, connection pooling (PgBouncer) |
| Ответ | Brotli/gzip сжатие, `ETag`/`Last-Modified`, `preload` вместо HTTP/2 push |
| Браузер | `defer`/`async` для JS, lazy loading, Service Workers |

## Примеры
```bash
# Смотреть детали рукопожатия и времени ответа
curl -vk https://example.com/ -o /dev/null
```

## Доп. теория
- HTTP/2 push считается устаревшим в современных браузерах; рекомендуют `preload`.
- HTTP/3 работает поверх QUIC (UDP) и включает TLS 1.3 внутри транспорта.
