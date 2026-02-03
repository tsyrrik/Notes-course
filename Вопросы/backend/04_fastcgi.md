## Вопрос: FastCGI

## Простой ответ
- Это язык общения между веб-сервером и приложением: сервер отдаёт запрос, приложение считает ответ.
- Долго живущие воркеры уже прогреты, поэтому быстрее чем старый CGI на каждый запрос.

## Ответ
- Клиент-серверный протокол между веб-сервером и приложением (php-fpm и др.), развитие CGI.
- Постоянные процессы вместо fork‑на‑запрос: быстрее и безопаснее.
- Общается по unix-socket или TCP, может быть на другом хосте.
- Можно запускать под отдельным пользователем / в chroot, параллельные воркеры.
- Работает с любым сервером, который поддерживает FastCGI (Nginx, Apache через mod_fastcgi и т.д.).

---

### Эволюция: CGI → FastCGI → альтернативы

Чтобы понять FastCGI, нужно знать историю. Изначально веб-серверы умели отдавать только статические файлы. Для динамического контента в 1993 году был создан **CGI (Common Gateway Interface)** — простой протокол, при котором веб-сервер для каждого запроса **запускает новый процесс** (fork + exec), передаёт ему данные через переменные окружения и stdin, получает ответ через stdout. После ответа процесс завершается.

Проблема CGI очевидна: fork + exec + инициализация интерпретатора на **каждый** запрос — это катастрофически медленно и неэффективно. Для PHP это означало: загрузить интерпретатор, прочитать php.ini, загрузить расширения, скомпилировать скрипт — и всё это ради одного запроса.

**FastCGI** (разработан Open Market в 1996 году) решает эту проблему: процессы-обработчики **запускаются один раз и живут долго**, принимая множество запросов последовательно. Это даёт огромный прирост производительности.

```text
=== CGI (классический) ===

Запрос 1: fork() → exec(php) → загрузка → выполнение → exit()
Запрос 2: fork() → exec(php) → загрузка → выполнение → exit()
Запрос 3: fork() → exec(php) → загрузка → выполнение → exit()
    ↑ каждый раз полная инициализация: ~50-200ms overhead

=== FastCGI ===

Старт: spawn php-fpm workers (1 раз)
Запрос 1: worker получает → выполнение → ответ → worker ждёт
Запрос 2: worker получает → выполнение → ответ → worker ждёт
Запрос 3: worker получает → выполнение → ответ → worker ждёт
    ↑ overhead на запрос: ~1-5ms (только передача данных по сокету)
```

### Сравнительная таблица протоколов

| Характеристика | CGI | FastCGI | mod_php | PHP-CLI |
|----------------|-----|---------|---------|---------|
| Процесс на запрос | Новый (fork+exec) | Persistent (pool) | Встроен в Apache | Одноразовый |
| Время инициализации | Высокое | Минимальное | Минимальное | Высокое |
| Изоляция | Полная | По запросу (в рамках процесса) | Нет (общий процесс с Apache) | Полная |
| Потребление памяти | Пиковое при нагрузке | Стабильное (пул) | Стабильное | — |
| Разные пользователи | Через suexec | Да (pool per user) | Нет (юзер Apache) | Да |
| Сетевая работа | Нет | Да (TCP/Unix) | Нет | Нет |
| Язык приложения | Любой | Любой | Только PHP | PHP |

### Протокол FastCGI: бинарный формат

FastCGI — это **бинарный** протокол (в отличие от текстового HTTP). Каждое сообщение состоит из заголовка (8 байт) и тела:

```text
Структура FastCGI-записи (record):
┌──────────────────────────────────────────┐
│ Version (1 byte)  = 1                    │
│ Type (1 byte)     = тип сообщения        │
│ Request ID (2 bytes)                     │
│ Content Length (2 bytes)                  │
│ Padding Length (1 byte)                   │
│ Reserved (1 byte)                        │
├──────────────────────────────────────────┤
│ Content Data (Content Length bytes)       │
│ Padding Data (Padding Length bytes)       │
└──────────────────────────────────────────┘

Типы сообщений:
  1  FCGI_BEGIN_REQUEST   — начало запроса
  2  FCGI_ABORT_REQUEST   — отмена запроса
  3  FCGI_END_REQUEST     — конец запроса
  4  FCGI_PARAMS          — параметры (key-value, аналог env в CGI)
  5  FCGI_STDIN           — тело запроса (POST data)
  6  FCGI_STDOUT          — тело ответа
  7  FCGI_STDERR          — ошибки
```

### Жизненный цикл FastCGI-запроса

```text
Веб-сервер (Nginx)                    FastCGI-приложение (PHP-FPM worker)
       |                                           |
       |── FCGI_BEGIN_REQUEST ───────────────────→ |  (role=RESPONDER)
       |── FCGI_PARAMS (SCRIPT_FILENAME, ...) ──→ |
       |── FCGI_PARAMS (пустой = конец) ─────────→|
       |── FCGI_STDIN (POST-данные) ─────────────→|
       |── FCGI_STDIN (пустой = конец) ──────────→|
       |                                           |
       |   ... PHP-FPM выполняет скрипт ...        |
       |                                           |
       |←── FCGI_STDOUT (HTTP-заголовки + тело) ──|
       |←── FCGI_STDERR (ошибки, в error.log) ────|
       |←── FCGI_END_REQUEST ─────────────────────|
       |                                           |
       |   Воркер готов к следующему запросу        |
```

### Параметры (FCGI_PARAMS), передаваемые веб-сервером

В FastCGI веб-сервер передаёт информацию о запросе через пары ключ-значение, аналогично переменным окружения в CGI:

```text
SCRIPT_FILENAME   = /var/www/site/public/index.php
QUERY_STRING      = page=2&sort=name
REQUEST_METHOD    = POST
CONTENT_TYPE      = application/json
CONTENT_LENGTH    = 128
SERVER_NAME       = site.com
SERVER_PORT       = 443
REMOTE_ADDR       = 192.168.1.100
REQUEST_URI       = /api/users?page=2
DOCUMENT_ROOT     = /var/www/site/public
HTTPS             = on
HTTP_HOST         = site.com
HTTP_USER_AGENT   = Mozilla/5.0 ...
HTTP_COOKIE       = session=abc123
```

В PHP эти параметры доступны через `$_SERVER`:
```php
$_SERVER['SCRIPT_FILENAME']  // /var/www/site/public/index.php
$_SERVER['REQUEST_METHOD']   // POST
$_SERVER['REMOTE_ADDR']      // 192.168.1.100
$_SERVER['HTTP_HOST']        // site.com
```

### Настройка связки в Nginx

```nginx
# Передача запросов в PHP-FPM через Unix-сокет
location ~ \.php$ {
    # Защита от выполнения произвольных файлов
    try_files $uri =404;

    fastcgi_pass unix:/run/php/php8.3-fpm.sock;

    # Обязательные параметры
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    include fastcgi_params;   # стандартный набор FCGI_PARAMS

    # Настройка буферов (ответ от PHP буферизируется перед отправкой клиенту)
    fastcgi_buffer_size 32k;       # буфер для первой части ответа (заголовки)
    fastcgi_buffers 8 16k;         # буферы для тела ответа
    fastcgi_busy_buffers_size 32k;

    # Таймауты
    fastcgi_connect_timeout 5s;    # таймаут подключения к FPM
    fastcgi_send_timeout 60s;      # таймаут отправки запроса в FPM
    fastcgi_read_timeout 60s;      # таймаут ожидания ответа от FPM
}
```

### Мультиплексирование запросов

FastCGI поддерживает мультиплексирование — несколько запросов по одному соединению (через Request ID). Однако на практике PHP-FPM **не использует** мультиплексирование: каждое соединение обрабатывает один запрос. Это связано с тем, что PHP выполняет код синхронно, и один воркер не может обрабатывать несколько запросов одновременно.

### FastCGI vs альтернативы

| Протокол | Описание | Где используется |
|----------|----------|-----------------|
| **FastCGI** | Бинарный, persistent workers | PHP-FPM, Python (flup), Ruby |
| **SCGI** | Упрощённый FastCGI (текстовые заголовки) | Python, редко |
| **uwsgi** | Бинарный протокол uWSGI | Python (Django, Flask) |
| **HTTP reverse proxy** | Стандартный HTTP | Node.js, Go, Java (Tomcat) |
| **ASGI** | Async Server Gateway Interface | Python (Django Channels, FastAPI) |
| **gRPC** | HTTP/2 + Protobuf | Микросервисы |

### Преимущества FastCGI для безопасности

FastCGI обеспечивает изоляцию на уровне процессов и пользователей:

```text
Nginx (user: nginx)
  │
  ├── Pool "site1" (user: site1, group: site1)
  │     └── Workers: uid=site1, chroot=/var/www/site1
  │
  ├── Pool "site2" (user: site2, group: site2)
  │     └── Workers: uid=site2, chroot=/var/www/site2
  │
  └── Pool "admin" (user: admin, group: admin)
        └── Workers: отдельный php.ini, другие лимиты
```

Каждый пул может иметь собственные настройки: `open_basedir`, `disable_functions`, лимиты памяти и времени. Это критически важно для shared-хостинга и мультитенантных приложений, где нужно изолировать пользователей друг от друга.

## Примеры
```nginx
# минимальная связка Nginx → PHP-FPM
location ~ \.php$ {
    fastcgi_pass unix:/run/php/php-fpm.sock;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

## Доп. теория
- FastCGI — протокол, а PHP‑FPM — конкретная реализация процессного менеджера для PHP.
- В реальных проектах FastCGI чаще всего используется именно для PHP‑FPM.
