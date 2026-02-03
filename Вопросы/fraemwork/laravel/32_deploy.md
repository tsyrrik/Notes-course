## Вопрос: Подготовка к продакшену
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Шаги подготовки к продакшену: оптимизации, переменные среды, права, миграции.

## Ответ
```bash
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
composer install --optimize-autoloader --no-dev
npm run production
php artisan migrate --force
chmod -R 775 storage bootstrap/cache
```
Очереди: `php artisan queue:restart` после деплоя; supervisor для воркеров.

## Примеры

1. `php artisan migrate --force`.
2. `php artisan queue:restart`.
3. `php artisan config:cache` после деплоя.

## Доп. теория

1. Деплой должен быть идемпотентным.
2. Env‑переменные не хранятся в репозитории.
