## Зачем вообще Docker

Без Docker ты ставишь зависимости прямо на систему — PHP, Postgres, Redis, Composer, Node… потом всё конфликтует, ломается или теряется при переезде.  
**Docker решает это**, создавая изолированные контейнеры, где у каждого приложения свой набор инструментов.  
Главное: **работает одинаково везде**, где есть Docker — хоть Linux, хоть Windows.

---

## Базовые сущности

|Термин|Что это|Аналогия|
|---|---|---|
|**Image (образ)**|Шаблон системы с нужными пакетами и настройками|Класс|
|**Container (контейнер)**|Запущенный экземпляр образа|Объект|
|**Network (сеть)**|Объединяет контейнеры и даёт им общие DNS-имена|Виртуальный LAN|
|**Volume (том)**|Хранилище данных, переживающее перезапуск контейнера|Внешний диск|
|**Registry (реестр)**|Репозиторий образов (например, Docker Hub)|Packagist|

---

## Контейнеры и образы

- **Образ** описывает, _что должно быть внутри_ (PHP, библиотеки, код).
    
- **Контейнер** — живая копия этого образа, со своими файлами и процессами.
    
- Можно создать десятки контейнеров из одного образа — каждый независим.
    

---

## Network

Контейнеры в одной сети видят друг друга по имени сервиса.  
Например, в `docker compose` по умолчанию создаётся общая сеть проекта.

```bash
# внутри контейнера app
ping db  # обращение к контейнеру "db"
```

DNS-сервер внутри Docker обычно `127.0.0.11`.

---

## Volume: данные и файлы

Volumes позволяют хранить данные вне контейнера, чтобы не потерять их при перезапуске.

- **Именованный (managed):**
    
    ```yaml
    volumes:
      - dbdata:/var/lib/postgresql/data
    ```
    
    Используй для баз данных — быстро и безопасно.
    
- **Bind (привязка):**
    
    ```yaml
    volumes:
      - ./app:/var/www/app
    ```
    
    Удобно при разработке (код на хосте сразу доступен в контейнере).  
    На macOS и Windows может быть медленно из-за шаринга FS.
    

⚠️ Старые данные в volume иногда вызывают конфликты. Если что-то "сломалось внезапно", попробуй удалить volume.

---

## Registry и зеркала

Docker Hub — основной реестр. Если он недоступен, добавь зеркало:

`/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["https://mirror.gcr.io", "https://registry-1.docker.io"]
}
```

Можно сделать то же через интерфейс Docker Desktop.

---

## CLI против Compose

- **`docker compose`** — современная версия (V2, встроена в Docker).
    
- **`docker-compose`** — старый Python-клиент (V1, не поддерживается).
    

**Разница между stop и down:**

|Команда|Что делает|
|---|---|
|`docker compose stop`|Останавливает контейнеры, но сеть и volume остаются.|
|`docker compose down`|Удаляет контейнеры и сеть. С `-v` удалит volume, с `--rmi local` — локальные образы.|

---

## Переменные окружения

- `.env` рядом с `compose.yml` — подставляет значения внутрь Compose.
    
- `environment:` — передаёт переменные внутрь контейнера.
    
- `env_file:` — подключает отдельный файл окружения.
    

Приоритет: **системные env > .env > значения в Compose**  
⚠️ Не коммить `.env` с секретами. Используй менеджеры секретов или `secrets:` в Compose.
Можно переопределять файл из которого брать енв

---

## Healthcheck и зависимости

Иногда база “готова”, но ещё не слушает порт.  
Добавь healthcheck и настрой зависимость:

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

---

## Volumes и производительность

- Для **баз данных** — именованные тома.
    
- Для **кода** — bind-mount, но на macOS/Windows может тормозить.
    
- Иногда выгодно держать отдельный volume под `vendor/` или `node_modules/`, чтобы не тянуть их каждый раз.
    

---

## Multistage-сборки (многостадийный Dockerfile)

Позволяют отделить стадию сборки от финального образа:

```dockerfile
# Сборка
FROM php:8.3-cli AS build
RUN apt-get update && apt-get install -y git unzip && rm -rf /var/lib/apt/lists/*
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --no-interaction
COPY . .

# Финальный образ
FROM php:8.3-cli AS runtime
WORKDIR /app
COPY --from=build /app /app
CMD ["php", "bin/console", "app:run"]
```

Преимущества:

- меньше размер,
    
- быстрее сборка,
    
- безопаснее (в runtime нет инструментов сборки).
    

---

## Кеширование слоёв

Docker кеширует шаги, если контент не изменился.  
Чтобы это работало:

1. Сначала копируй только `composer.json`/`package.json` и ставь зависимости.
    
2. Потом копируй код.
    
3. Не забывай чистить `apt` в том же слое:
    

```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

---

## ARG vs ENV

|Тип|Когда доступен|Пример использования|
|---|---|---|
|`ARG`|Только во время сборки|Версия PHP или бинарника|
|`ENV`|Во время выполнения|`APP_ENV=prod`, `PHP_MEMORY_LIMIT=512M`|

`ARG` не сохраняется в финальный образ, `ENV` — да.

---

## Полезные мелочи

- Порты прокидывай из `.env`, чтобы легко менять при конфликтах.
    
- Не держи базы данных в bind-mount’ах — это медленно.
    
- Для PHP-образов ставь расширения через [mlocati/php-extension-installer](https://github.com/mlocati/docker-php-extension-installer):
    
    ```dockerfile
    COPY --from=mlocati/php-extension-installer:2.7.5 /usr/bin/install-php-extensions /usr/local/bin/
    RUN install-php-extensions amqp intl opcache pdo_pgsql redis zip
    ```
    
- Docker относится к движению **Infrastructure as Code**.  
    Его можно проверять на уязвимости, например инструментом [KICS](https://habr.com/ru/articles/889120/#kics).
    

---

## Замечание о производительности

- Базы данных внутри Docker **чуть медленнее**, чем на хосте — из-за виртуализации FS и сетевого слоя.
    
- Для PHP, Node и других приложений это незаметно.
    
- Используй именованные тома и не ставь Xdebug в прод, если хочешь сэкономить память.
    
