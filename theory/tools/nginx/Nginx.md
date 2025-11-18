## Где что лежит

- `/etc/nginx/` — вся конфигурация.
    
    - `nginx.conf` — общий конфиг, воркеры, лог-форматы.
        
    - `conf.d/*.conf` — доп. конфиги (часто пустые).
        
    - `sites-available/` — лежат серверы (вирт. хосты).
        
    - `sites-enabled/` — активные сайты (симлинки на available).
        
    - `snippets/` — куски, типа `ssl-params.conf`, `gzip.conf`.
        
- Логи: `/var/log/nginx/access.log`, `/var/log/nginx/error.log`
    
- Проверка и перезагрузка:
    
    ```bash
    nginx -t # проверка что конфиг синтаксически корректен и нжинкс сможет стартовать с ним
    nginx -s reload # говорим мастер процессу нжинкса перечитать конфиг
    ```
    

---

## Базовый каркас `nginx.conf` 

Обычно достаточно дефолтного конфига, если только ты не девопс или у тебя не какие-то высокие требования к приложению

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events { worker_connections 4096; }

http {
  include       mime.types;
  default_type  application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log warn;

  sendfile on;
  keepalive_timeout 65;
  server_tokens off;

  # Gzip — включи и живи спокойнее
  gzip on; gzip_types text/plain text/css application/json application/javascript application/xml;
  gzip_min_length 1024;

  include /etc/nginx/sites-enabled/*;
}
```

---

## Один домен: фронт + бэк (SPA + API) + WebSocket

`/etc/nginx/sites-available/app.example.com`:

```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

upstream backend_api {
  server 127.0.0.1:9000;  # ваш backend (symfony/laravel/rr/swoole/frankenphp)
  keepalive 32;
}

server {
  listen 80;
  server_name app.example.com;
  return 301 https://$host$request_uri;  # редиректим сразу на https
}

server {
  listen 443 ssl http2;
  server_name app.example.com;

  # SSL — сюда позже certbot подставит свои пути
  ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
  include /etc/letsencrypt/options-ssl-nginx.conf; # best-practice от certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

  # Логи отдельно по проекту
  access_log /var/log/nginx/app_access.log main;
  error_log  /var/log/nginx/app_error.log warn;

  # Корень для фронта (собранный SPA)
  root /var/www/frontend/dist;
  index index.html;

  # Статика фронта отдаем напрямую, долго кэшируем
  location /assets/ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    try_files $uri =404;
  }

  # SPA-роутинг: все неизвестные пути — на index.html
  location / {
    try_files $uri $uri/ /index.html;
  }

  # API — проксируем на бэкенд
  location /api/ {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
    proxy_read_timeout 60s;
    proxy_send_timeout 60s;

    proxy_pass http://backend_api;
  }

  # WebSocket (например, /ws) — важны Upgrade/Connection
  location /ws/ {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    proxy_pass http://backend_api;
  }

  # CORS для API (минимально адекватный вариант)
  location /api/ {
    if ($request_method = OPTIONS) {
      add_header Access-Control-Allow-Origin  "$http_origin" always;
      add_header Access-Control-Allow-Methods "GET,POST,PUT,PATCH,DELETE,OPTIONS" always;
      add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Requested-With" always;
      add_header Access-Control-Allow-Credentials "true" always;
      add_header Content-Length 0;
      add_header Content-Type text/plain;
      return 204;
    }

    add_header Access-Control-Allow-Origin  "$http_origin" always;
    add_header Access-Control-Allow-Credentials "true" always;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;

    proxy_pass http://backend_api;
  }

  # Простейшие security-заголовки
  add_header X-Frame-Options DENY always;
  add_header X-Content-Type-Options nosniff always;
  add_header Referrer-Policy no-referrer-when-downgrade always;
}
```

Активируем сайт и перегружаем:

```bash
ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/
nginx -t && nginx -s reload
```

---

## Certbot: HTTPS и автопродление

По умолчанию наш сервак работает по http, чтоб добавить https нужны сертификаты
Проще всего их сгенерить с помощью certbot утилиты

Быстрый путь:

```bash
apt-get install -y certbot python3-certbot-nginx
certbot --nginx -d app.example.com --redirect --agree-tos -m you@example.com
```

- Certbot сам пропишет `ssl_certificate` и все что для них нужно.
    
- Автопродление: ставится **systemd timer** (`systemctl list-timers | grep certbot`), руками ничего крутить не нужно.
    
- Проверка:
    
    ```bash
    certbot renew --dry-run
    ```
    

---

## CORS коротко

- Лучше на бэкенде, но иногда проще закрыть в nginx.
    
- Для **простых** запросов хватит `Access-Control-Allow-Origin` и `Allow-Credentials`.
    
- **Preflight (OPTIONS)** обязательно отвечать 204, иначе фронт ругается.
    

---

## Разруливание фронта и бэка на одном домене

- `/` — отдаём SPA, `try_files ... /index.html`.
    
- `/api/` — прокси в бекенд.
    
- `/ws/` — WebSocket на тот же бекенд.
    
- Можно добавить второй upstream для php-fpm, если бекенд — именно PHP-скрипты:
    
    ```nginx
    location ~ \.php$ {
      include fastcgi_params;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      fastcgi_pass unix:/run/php/php-fpm.sock;
    }
    ```
    
    но для SPA+API чаще удобнее HTTP-бэкенд (RR/Swoole/FrankenPHP/Node).
    

---

## Полезные фишки “на вырост”

- **Rate limiting** (чтобы DDoSили кого-то другого):
    
    ```nginx
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    location /api/ {
      limit_req zone=api_limit burst=20 nodelay;
      ...
    }
    ```
    
- **Proxy buffering** для больших ответов:
    
    ```nginx
    proxy_buffering on;
    proxy_buffers 8 32k;
    proxy_busy_buffers_size 64k;
    ```
    
- **Real IP за прокси/Cloudflare**:
    
    ```nginx
    set_real_ip_from 0.0.0.0/0;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    ```
    
- **Health-checks**:
    
    ```nginx
    location = /healthz { return 200 'ok'; add_header Content-Type text/plain; }
    ```
    
- **Логи по проектам**: не смешивай всё в общий `access.log`.
    

---

## Логи и ротация

- Смотри ошибки:
    
    ```bash
    tail -f /var/log/nginx/error.log
    ```
    
- Ротация обычно настроена через `/etc/logrotate.d/nginx`. Не трогай, если не знаешь, что делаешь.
    

---

## Частые грабли

- **Порядок location’ов**: точные `location =` и префиксные идут до регекспов `~`.
    
- **CORS preflight**: забыл OPTIONS → фронт орёт, ты грустишь.
    
- **WebSocket**: без `Upgrade/Connection` не полетит.
    
- **SSL**: автообновление есть, но перезагрузка после обновления иногда нужна (certbot делает `--deploy-hook` сам).
    
- **try_files**: для SPA всегда добавляй `/index.html` в конце.
    

---

## Итого

- Конфиги держи в `sites-available` и линкай в `sites-enabled`.
    
- Один домен легко тащит фронт, API и WS, если правильно настроить `location`.
    
- HTTPS — через certbot, автопродление по таймеру.
    
- CORS руками в nginx — ок, но лучше закрывать на бэке.
    
- Проверяй `nginx -t` и живи дольше остальных.
    

