## Вопрос: Веб-сервер и PHP: NGINX+PHP-FPM vs Apache+mod_php

## Простой ответ
- Nginx+FPM: быстрый фронт и пул PHP-воркеров отдельно; хорошо под нагрузкой и статику.
- Apache+mod_php: проще поднять, PHP встроен в процессы сервера, но тяжелее и хуже масштаб под пиковые запросы.

## Ответ
### NGINX + PHP-FPM
- Nginx принимает HTTP, статикой отвечает сам, PHP-запросы отправляет в php-fpm (FastCGI).
- Пока php-fpm считает, Nginx обслуживает другие запросы — высокая масштабируемость.
- Нужно настроить связку (unix-сокет/TCP), число воркеров и буферы.

### Apache + mod_php
- Apache сам загружает PHP как модуль; процесс/поток обрабатывает запрос и выполняет PHP.
- Простой старт, но тяжёлые процессы и хуже масштабирование под нагрузкой.

Когда что: Nginx+FPM — для нагруженных проектов/статик+PHP; Apache+mod_php — для простого/legacy окружения.

---

### Архитектурные модели: событийная vs процессная

Ключевое архитектурное различие между Nginx и Apache лежит в модели обработки соединений. Nginx использует **событийно-ориентированную** (event-driven) асинхронную модель: один рабочий процесс обслуживает тысячи соединений через системные вызовы `epoll` (Linux) / `kqueue` (macOS/FreeBSD). Apache по умолчанию работает в модели **prefork** (один процесс на соединение) или **worker/event** (потоки), но при использовании mod_php ограничен prefork, так как PHP не является потокобезопасным (non-thread-safe).

```text
=== Nginx (event-driven) ===

Master process
  ├── Worker 1 (1 поток, тысячи соединений через epoll)
  ├── Worker 2 (1 поток, тысячи соединений)
  └── Worker N (обычно = кол-во CPU ядер)

Каждый worker обрабатывает соединения неблокирующе:
  → принял данные → отправил в FastCGI → переключился на другое соединение
  → получил ответ от FastCGI → отправил клиенту → освободился

=== Apache prefork (mod_php) ===

Master process
  ├── Child 1 (1 процесс = 1 соединение, PHP встроен)
  ├── Child 2 (1 процесс = 1 соединение, PHP встроен)
  ├── ...
  └── Child N (MaxRequestWorkers, обычно 150-256)

Каждый child-процесс полностью занят одним запросом,
пока тот не завершится (включая ожидание БД, I/O).
```

### Потребление памяти

Это главная причина, почему Nginx+PHP-FPM лучше масштабируется:

| Метрика | Nginx + PHP-FPM | Apache + mod_php (prefork) |
|---------|-----------------|---------------------------|
| Память на процесс Nginx | несколько МБ | — |
| Память на процесс Apache | — | десятки МБ (с PHP) |
| Память на PHP-FPM worker | десятки МБ | Встроен в Apache‑процесс |
| 100 одновременных запросов | зависит от настроек FPM | зависит от MPM и модулей |
| 1000 keep-alive соединений | небольшой overhead | очень большой overhead |
| Статика (CSS/JS/img) | Nginx отдаёт без PHP | Apache держит тяжёлый процесс |

Критическая разница: в Apache+mod_php каждое keep-alive соединение держит целый процесс с загруженным PHP, даже если клиент просто ждёт. В Nginx keep-alive соединения обслуживаются событийным циклом с минимальным потреблением памяти, а PHP-FPM воркеры заняты **только** во время реального выполнения PHP-кода.

### Конфигурация Nginx + PHP-FPM

```nginx
# /etc/nginx/sites-available/mysite.conf
server {
    listen 80;
    server_name mysite.com;
    root /var/www/mysite/public;
    index index.php;

    # Статика — Nginx отдаёт сам, без участия PHP
    location ~* \.(css|js|jpg|png|gif|ico|woff2|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;    # не засоряем логи
        try_files $uri =404;
    }

    # Фронт-контроллер (Laravel/Symfony)
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Передача PHP в FPM
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;  # или 127.0.0.1:9000
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;

        # Таймауты и буферы
        fastcgi_read_timeout 60s;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
    }
}
```

```ini
; /etc/php/8.3/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data

; Способ связи: unix-сокет быстрее TCP на localhost
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data

; Управление пулом воркеров
pm = dynamic                ; static | dynamic | ondemand
pm.max_children = 50        ; макс. воркеров
pm.start_servers = 10       ; при старте
pm.min_spare_servers = 5    ; минимум свободных
pm.max_spare_servers = 20   ; максимум свободных
pm.max_requests = 500       ; перезапуск воркера после N запросов (защита от утечек)

; Логирование медленных запросов
slowlog = /var/log/php-fpm/slow.log
request_slowlog_timeout = 5s
request_terminate_timeout = 30s
```

### Конфигурация Apache + mod_php

```apache
# /etc/apache2/sites-available/mysite.conf
<VirtualHost *:80>
    ServerName mysite.com
    DocumentRoot /var/www/mysite/public

    <Directory /var/www/mysite/public>
        AllowOverride All        # Разрешает .htaccess (дополнительная нагрузка!)
        Require all granted
    </Directory>

    # PHP уже встроен через mod_php — обрабатывается автоматически
    <FilesMatch \.php$>
        SetHandler application/x-httpd-php
    </FilesMatch>
</VirtualHost>
```

```apache
# /etc/apache2/mods-available/mpm_prefork.conf
<IfModule mpm_prefork_module>
    StartServers             5
    MinSpareServers          5
    MaxSpareServers         10
    MaxRequestWorkers      150   # макс. одновременных соединений
    MaxConnectionsPerChild 500   # перезапуск после N запросов
</IfModule>
```

### Обработка запроса: пошаговое сравнение

```text
=== Nginx + PHP-FPM ===

1. Клиент → TCP-соединение → Nginx (epoll/kqueue)
2. Nginx парсит HTTP-запрос
3. location match:
   a) Статика → sendfile() → клиент (PHP не участвует)
   b) PHP → FastCGI → php-fpm.sock
4. FPM master → свободный worker
5. Worker выполняет PHP-код
6. Ответ: Worker → FPM → Nginx → клиент
7. Worker возвращается в пул, Nginx обслуживает другие соединения

=== Apache + mod_php ===

1. Клиент → TCP-соединение → Apache
2. Apache форкает/выбирает child-процесс
3. Child занят этим соединением до завершения
4. Apache читает .htaccess (на каждый запрос, в каждой директории!)
5. mod_php исполняет PHP прямо в child-процессе
6. Ответ → клиент
7. Child может обслужить следующий запрос (или остаётся занят при keep-alive)
```

### Unix-сокет vs TCP для связи Nginx↔PHP-FPM

| Характеристика | Unix Socket | TCP (127.0.0.1:9000) |
|----------------|-------------|----------------------|
| Скорость | Быстрее (~5-10%) | Чуть медленнее (TCP overhead) |
| Расположение | Только localhost | Может быть удалённый сервер |
| Настройка | Права на файл сокета | Firewall, порты |
| Масштабирование | Один хост | Несколько серверов PHP-FPM |
| Отладка | Сложнее (нет tcpdump) | Проще (tcpdump, telnet) |

### Режимы пула PHP-FPM: static vs dynamic vs ondemand

```text
pm = static
├── Всегда pm.max_children воркеров в памяти
├── Быстрый отклик (нет задержки на форк)
└── Высокое потребление памяти (даже при простое)

pm = dynamic (рекомендуется для большинства)
├── Между min_spare и max_spare воркеров в зависимости от нагрузки
├── Баланс между скоростью и памятью
└── Небольшая задержка при всплеске нагрузки

pm = ondemand
├── 0 воркеров в простое, форкает по запросу
├── Минимальное потребление памяти
└── Задержка на холодный старт (подходит для dev/малонагруженных сайтов)
```

### Когда что выбирать: итоговая таблица

| Критерий | Nginx + PHP-FPM | Apache + mod_php |
|----------|-----------------|------------------|
| Высоконагруженный проект | **Да** | Нет |
| Много статики (CDN отсутствует) | **Да** (sendfile) | Тяжело |
| Микросервисы, Docker | **Да** (лёгкий, предсказуемый) | Избыточен |
| Shared-хостинг с .htaccess | Нет | **Да** |
| Быстрый старт без настройки | Нет | **Да** (apt install libapache2-mod-php) |
| Legacy-приложения с .htaccess | Нет (нужна конвертация в nginx rewrite) | **Да** |
| Горизонтальное масштабирование PHP | **Да** (FPM на отдельном сервере) | Сложнее |
| Совместимость с mod_rewrite | Нет (свой синтаксис) | **Да** |

### Гибридная схема: Nginx как reverse proxy перед Apache

В некоторых legacy-проектах используют комбинацию: Nginx на фронте принимает соединения и отдаёт статику, а динамику проксирует в Apache+mod_php:

```text
Клиент → Nginx (порт 80/443)
            ├── статика → sendfile
            └── *.php → proxy_pass http://127.0.0.1:8080 → Apache+mod_php
```

Это позволяет сохранить .htaccess-совместимость и при этом получить преимущества Nginx для статики и SSL-терминации.

## Примеры
```bash
# Проверить, что PHP обрабатывается через FPM
curl -I https://site.com/index.php
```

## Доп. теория
- Современный Apache чаще используют с `mpm_event` и `php-fpm` через `proxy_fcgi`, чтобы избежать ограничений `mod_php`.
- `.htaccess` удобен, но дорог по производительности: его поиск идёт на каждый запрос.
