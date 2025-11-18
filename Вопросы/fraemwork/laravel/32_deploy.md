# Подготовка к продакшену

Простыми словами: Шаги подготовки к продакшену: оптимизации, переменные среды, права, миграции.
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
