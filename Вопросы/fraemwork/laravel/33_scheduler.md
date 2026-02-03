## Вопрос: Планировщик задач (Scheduler)
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Scheduler — cron в коде, один системный cron запускает все запланированные задачи.

## Ответ
Код расписания в `app/Console/Kernel.php`, запускается одним cron `* * * * * php artisan schedule:run`.
```php
protected function schedule(Schedule $schedule) {
    $schedule->command('reports:generate')->hourly();
    $schedule->job(new ClearOldLogsJob)->dailyAt('03:00');
    $schedule->call(fn() => cleanup())->weekdays()->at('09:00');
}
```
Отличие от очередей: scheduler запускает задачи по расписанию, очереди — асинхронное выполнение задач по мере поступления (часто вместе: scheduler диспатчит jobs).

## Примеры

1. `$schedule->command('reports:daily')->daily()`.
2. `$schedule->job(new SyncJob)->hourly()`.
3. Cron: `* * * * * php artisan schedule:run`.

## Доп. теория

1. Один cron запускает весь планировщик.
2. `withoutOverlapping()` защищает от дублей.
