## Вопрос: Очереди в Laravel
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Очереди запускают тяжёлые задачи (job) в фоне, чтобы не тормозить HTTP.

## Ответ
Отложенное выполнение тяжёлых задач.
```bash
php artisan make:job SendEmailJob
```
```php
class SendEmailJob implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    public function __construct(public User $user, public string $message) {}
    public function handle(MailService $mailService) { $mailService->send($this->user->email, $this->message); }
    public function failed(Throwable $e) { Log::error('Email failed', ['user_id'=>$this->user->id,'error'=>$e->getMessage()]); }
}
SendEmailJob::dispatch($user,'Welcome')->delay(now()->addHours(2))->onQueue('high');
```

## Настройка и worker
```php
// config/queue.php connections (database/redis)
php artisan queue:table && php artisan migrate
php artisan queue:work --queue=high,default --timeout=60 --tries=3
```
Supervisor (прод):
```
[program:laravel-worker]
command=php /path/to/project/artisan queue:work --sleep=3 --tries=3
numprocs=8
```
