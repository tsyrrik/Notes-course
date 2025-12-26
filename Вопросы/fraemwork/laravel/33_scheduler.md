## Вопрос: Планировщик задач (Scheduler)
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

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
