## Вопрос: Symfony Flex
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Flex управляет зависимостями и "рецептами" — автогенерирует конфиги, папки, пакеты при установке через Composer.

## Ответ

### Что такое Symfony Flex

Symfony Flex -- это плагин для Composer, который расширяет стандартный процесс установки пакетов. Он подключается автоматически при создании проекта через `symfony/skeleton` и перехватывает операции `composer require` / `composer remove`. Главная задача Flex -- автоматизация конфигурации пакетов с помощью так называемых **рецептов** (recipes).

### Как работают рецепты

Когда вы устанавливаете пакет, Flex обращается к серверу рецептов (по умолчанию `https://repo.symfony.com`) и скачивает инструкции для этого пакета. Рецепт может выполнять следующие действия:

- Создавать конфигурационные файлы в `config/packages/`
- Добавлять переменные окружения в `.env`
- Создавать директории (например, `templates/`, `translations/`)
- Регистрировать бандл в `config/bundles.php`
- Добавлять строки в `.gitignore`
- Выполнять другие подготовительные действия

Существует два репозитория рецептов:
- **symfony/recipes** -- официальные, проверенные рецепты (contrib от основной команды)
- **symfony/recipes-contrib** -- рецепты от сообщества

```bash
# Создание минимального проекта
composer create-project symfony/skeleton myapp

# Установка ORM (Flex подтянет doctrine-bundle, orm-pack, создаст config/packages/doctrine.yaml)
composer require orm

# Установка Twig (Flex создаст templates/base.html.twig, config/packages/twig.yaml)
composer require twig

# Полный веб-стек одной командой
composer require webapp
```

### Алиасы пакетов

Flex предоставляет короткие **алиасы** для часто используемых пакетов, чтобы не запоминать полные имена:

| Алиас | Пакет |
|-------|-------|
| `orm` | `symfony/orm-pack` |
| `twig` | `symfony/twig-bundle` |
| `security` | `symfony/security-bundle` |
| `mailer` | `symfony/mailer` |
| `debug` | `symfony/debug-bundle` |
| `test` | `symfony/test-pack` |
| `log` | `symfony/monolog-bundle` |
| `maker` | `symfony/maker-bundle` |

### Удаление пакетов

При удалении пакета через `composer remove` Flex выполняет «unrecipe» -- откатывает изменения рецепта: удаляет созданные конфигурационные файлы, убирает бандл из `config/bundles.php` и т.д.

```bash
composer remove orm   # Flex удалит config/packages/doctrine.yaml и связанные файлы
```

### Packs -- мета-пакеты

Symfony использует понятие **pack** -- мета-пакет, который объединяет несколько связанных пакетов. Например, `symfony/orm-pack` подтягивает `doctrine/orm`, `doctrine/doctrine-bundle`, `doctrine/doctrine-migrations-bundle`. После установки pack можно «распаковать» (unpack), чтобы зависимости стали прямыми:

```bash
composer require orm           # Устанавливает orm-pack
composer unpack symfony/orm-pack  # Заменяет мета-пакет на прямые зависимости в composer.json
```

### Практические советы

- Всегда используйте `symfony/skeleton` для новых проектов -- это минимальная отправная точка, и вы добавляете только то, что нужно
- Проверяйте, какие файлы создал рецепт, через `git diff` после установки пакета
- Если рецепт не создал нужный конфиг или вы случайно удалили его, выполните `composer recipes:install vendor/package --force` для повторной установки рецепта
- Команда `composer recipes` покажет список всех установленных рецептов и их статус
- В Symfony 7 Flex остаётся стандартным способом управления зависимостями и конфигурацией проекта

## Примеры

1. `composer require orm` добавляет Doctrine и создает `config/packages/doctrine.yaml`.
2. `composer require twig` добавляет шаблоны и `templates/base.html.twig`.
3. `composer recipes` показывает установленные рецепты.

## Доп. теория

1. Рецепты — это инструкции по настройке, а не сами пакеты.
2. `composer unpack` делает зависимости явными в `composer.json`.
