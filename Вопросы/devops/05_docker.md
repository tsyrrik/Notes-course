# Docker
Изоляция зависимостей в контейнерах: работает одинаково на любой машине с Docker.

## Базовые сущности
| Термин | Что это | Аналогия |
| --- | --- | --- |
| Image | Шаблон системы с пакетами/кодом | Класс |
| Container | Запущенный экземпляр образа | Объект |
| Network | Общая сеть/DNS между контейнерами | Виртуальный LAN |
| Volume | Данные, переживающие перезапуск | Внешний диск |
| Registry | Репозиторий образов (Docker Hub) | Packagist |

## Образы и контейнеры
- Образ описывает что внутри (PHP, код, либы).
- Контейнер — живая копия образа с собственными файлами/процессами.
- Несколько контейнеров из одного образа независимы.

## Network
- В одной сети контейнеры пингуют друг друга по имени сервиса (`ping db` внутри `app`).
- В compose общая сеть создаётся по умолчанию, DNS внутри: `127.0.0.11`.

## Volume: данные и файлы
- Именованный (managed): `volumes: [dbdata:/var/lib/postgresql/data]` — быстрый и удобен для БД.
- Bind: `volumes: ["./app:/var/www/app"]` — для разработки (код на хосте сразу виден), на macOS/Windows медленнее из-за шаринга FS.
- Старые данные в томах иногда ломают окружение — чистка volume может помочь.

## Registry и зеркала
- Docker Hub — основной; при проблемах добавь зеркала:
```json
// /etc/docker/daemon.json
{ "registry-mirrors": ["https://mirror.gcr.io", "https://registry-1.docker.io"] }
```

## CLI vs Compose
- `docker compose` (V2) — актуальный; `docker-compose` (V1) — устаревший.
- Stop vs Down:
| Команда | Что делает |
| --- | --- |
| `docker compose stop` | Останавливает контейнеры, сеть/volume остаются |
| `docker compose down` | Удаляет контейнеры и сеть; `-v` удалит volume, `--rmi local` — образы |

## Переменные окружения
- `.env` рядом с compose: подставляется в `compose.yml`.
- `environment:` — внутрь контейнера; `env_file:` — подключить файл.
- Приоритет: системные env > `.env` > значения в compose. Не коммить секреты в `.env`; используй secrets/менеджеры.

## Healthcheck и зависимости
```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
  db:
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      retries: 5
```

## Производительность томов
- БД — именованные тома; код — bind (в dev); иногда выделяют отдельный volume под `vendor/` или `node_modules/`.

## Multistage Dockerfile
Отделяет сборку от рантайма, делает образ меньше/без build-инструментов.
```dockerfile
# build
FROM php:8.3-cli AS build
RUN apt-get update && apt-get install -y git unzip && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction
COPY . .

# runtime
FROM php:8.3-cli AS runtime
WORKDIR /app
COPY --from=build /app /app
CMD ["php", "bin/console", "app:run"]
```

## Кеширование слоёв
1) Сначала копируй только `composer.json/package.json` и ставь зависимости.  
2) Потом копируй код.  
3) Чисти `apt` в том же слое: `RUN apt-get update && ... && rm -rf /var/lib/apt/lists/*`.

## ARG vs ENV
| Тип | Когда доступен | Пример |
| --- | --- | --- |
| ARG | Во время сборки | версия PHP/бинарника |
| ENV | Во время выполнения | `APP_ENV=prod`, `PHP_MEMORY_LIMIT=512M` |
ARG не сохраняется в финальный образ, ENV — да.

## Полезные мелочи
- Пробрасывай порты из `.env`, базы держи на именованных томах.
- Не ставь Xdebug в прод.
- PHP-расширения через mlocati installer:
```dockerfile
COPY --from=mlocati/php-extension-installer:2.7.5 /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions amqp intl opcache pdo_pgsql redis zip
```
- Docker — часть IaC, проверяй на уязвимости (KICS и т.п.).

## bind-mount vs именованный том (коротко)
| Нужно | Что выбрать | Почему |
| --- | --- | --- |
| Разработка, горячие изменения кода | bind-mount | изменения сразу видны |
| Прод, данные/БД | именованный том | скорость, стабильность, независимость от путей |
| Кэш/временные | именованный том | не нужен доступ с хоста |
