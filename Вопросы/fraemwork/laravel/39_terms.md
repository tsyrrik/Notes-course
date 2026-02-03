## Вопрос: Краткие определения
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Короткие определения ключевых терминов Laravel.

## Ответ
- Service Provider — точка входа для регистрации зависимостей/конфигов/событий.
- Eloquent — ORM (ActiveRecord): модели = таблицы, связи, события, касты.
- Query Builder — объектный построитель SQL, легче/быстрее Eloquent, возвращает массивы/коллекции.
- Scheduler — описание cron-задач в коде, запускается `php artisan schedule:run`.
- Rate Limiter — ограничитель запросов (throttle middleware).
- HTTP Client — обёртка над Guzzle c retry, fake, pool.
- Guzzle — базовый HTTP-клиент PHP, на нём построен Laravel HTTP Client.
- Event/Listener — событие (факт) и обработчик реакции.
- Queue/Job — отложенные задачи, worker обрабатывает.
- Middleware — фильтр между клиентом и контроллером.
- Facade — статический интерфейс к сервису в контейнере.
- Policy — авторизация действий над моделью; Gates — одноразовые проверки.
- Resource/ResourceCollection — трансформеры API JSON.
- Storage — универсальный интерфейс работы с файлами (local/public/S3).

## Примеры

1. Facade: `Cache::get()`.
2. Service Container: `app()->make()`.
3. Eloquent vs Query Builder как термины.

## Доп. теория

1. Глоссарий помогает согласовать терминологию команды.
2. Важно различать ORM, Query Builder и Facade.
