## Вопрос: Конфигурация и окружения
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
настройки лежат в `config/` и зависят от окружения (`.env`, `.env.local`). Параметры доступны через `%env(VAR)%` и `parameters`.

## Ответ

### Окружения (Environments)

Symfony поддерживает несколько окружений, каждое из которых определяет набор конфигураций, уровень отладки и поведение приложения. По умолчанию доступны три окружения: `dev`, `prod` и `test`. Окружение определяется переменной `APP_ENV`.

- **dev** -- полная отладка, Web Profiler, подробные ошибки, без кэширования шаблонов
- **prod** -- оптимизированное для производительности, минимальное логирование, скомпилированный контейнер
- **test** -- для автоматических тестов, свои параметры БД, отключены лишние сервисы

### Файлы .env и приоритет загрузки

Symfony использует компонент Dotenv для загрузки переменных окружения. Файлы загружаются в следующем порядке (каждый следующий перезаписывает предыдущие):

1. `.env` -- значения по умолчанию, коммитится в git
2. `.env.local` -- локальные переопределения, НЕ коммитится (в `.gitignore`)
3. `.env.{APP_ENV}` -- значения для конкретного окружения (`.env.test`)
4. `.env.{APP_ENV}.local` -- локальные переопределения для окружения

```env
# .env (коммитится в git — только дефолтные/несекретные значения)
APP_ENV=dev
APP_DEBUG=1
APP_SECRET=change_me_in_env_local
DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
MESSENGER_TRANSPORT_DSN=doctrine://default?auto_setup=0
MAILER_DSN=null://null
```

```env
# .env.local (НЕ коммитится — реальные секреты и настройки)
APP_SECRET=a1b2c3d4e5f6
DATABASE_URL="postgresql://myuser:realpass@db.server:5432/production_db"
```

```env
# .env.test (коммитится — настройки для тестового окружения)
KERNEL_CLASS='App\Kernel'
APP_SECRET='$ecretf0teletest'
DATABASE_URL="sqlite:///%kernel.project_dir%/var/test.db"
```

### Конфигурация пакетов по окружениям

Конфигурационные файлы в `config/packages/` могут быть специфичными для окружения:

```
config/
├── packages/
│   ├── cache.yaml              # Общая конфигурация кэша
│   ├── doctrine.yaml           # Общая конфигурация Doctrine
│   ├── framework.yaml          # Общие настройки фреймворка
│   ├── twig.yaml
│   ├── dev/                    # Только для dev-окружения
│   │   ├── debug.yaml
│   │   ├── monolog.yaml
│   │   └── web_profiler.yaml
│   ├── prod/                   # Только для prod
│   │   └── monolog.yaml
│   └── test/                   # Только для test
│       └── framework.yaml
├── routes/
│   └── dev/
│       └── web_profiler.yaml   # Роуты профайлера только в dev
├── bundles.php
├── routes.yaml
└── services.yaml
```

### Использование переменных окружения в конфигах

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        url: '%env(DATABASE_URL)%'    # Считывает из переменной окружения

# Процессоры для преобразования значений
parameters:
    redis_port: '%env(int:REDIS_PORT)%'          # Приведение к int
    feature_flag: '%env(bool:FEATURE_ENABLED)%'   # Приведение к bool
    db_password: '%env(base64:DB_PASSWORD)%'      # Base64 декодирование
    config: '%env(json:APP_CONFIG)%'              # Парсинг JSON
    secret: '%env(file:SECRET_FILE)%'             # Чтение из файла
    url_path: '%env(key:path:url:DATABASE_URL)%'  # Извлечение части URL
```

### Symfony Secrets (Vault)

Для хранения секретов в production Symfony предоставляет встроенный vault, который шифрует секреты и позволяет коммитить их в git:

```bash
# Создать ключи шифрования
php bin/console secrets:generate-keys

# Добавить секрет
php bin/console secrets:set DATABASE_PASSWORD
# Вводите значение интерактивно

# Для prod-окружения
php bin/console secrets:set DATABASE_PASSWORD --env=prod

# Список секретов
php bin/console secrets:list --reveal

# Удалить секрет
php bin/console secrets:remove DATABASE_PASSWORD
```

Секреты хранятся зашифрованными в `config/secrets/{env}/` и доступны как обычные `%env()%` параметры.

### Параметры контейнера

Помимо env-переменных, Symfony поддерживает параметры контейнера (статические значения):

```yaml
# config/services.yaml
parameters:
    app.admin_email: 'admin@example.com'
    app.items_per_page: 20
    app.supported_locales: ['ru', 'en', 'de']

services:
    App\Service\PaginationService:
        arguments:
            $perPage: '%app.items_per_page%'
```

### Отладка конфигурации

```bash
# Показать итоговую конфигурацию пакета
php bin/console debug:config framework

# Показать дефолтную конфигурацию
php bin/console config:dump-reference framework

# Показать все параметры контейнера
php bin/console debug:container --parameters

# Показать все env-переменные
php bin/console debug:container --env-vars
```

### Практические советы

- Никогда не коммитьте `.env.local` -- он содержит реальные секреты
- В `.env` храните только дефолтные значения и примеры (шаблон для разработчиков)
- Для production используйте реальные переменные окружения сервера (Docker, Kubernetes) или Symfony Secrets
- Переменная `APP_DEBUG` отключает кэширование контейнера и шаблонов -- на production она должна быть `0`
- Используйте `%env(resolve:...)%` для рекурсивного разрешения параметров внутри env-переменных
- В Symfony 7 env-процессоры (int, bool, json, file и др.) позволяют типизировать параметры прямо в конфиге

## Примеры

1. `DATABASE_URL` в `.env.local`, а дефолт в `.env`.
2. Конфиг для dev: `config/packages/dev/web_profiler.yaml`.
3. Параметр `%env(int:REDIS_PORT)%` для приведения типа.

## Доп. теория

1. Порядок загрузки `.env` важен для переопределений.
2. В prod предпочтительны реальные env‑переменные или Secrets.
