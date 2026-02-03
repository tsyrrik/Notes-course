## Вопрос: Очереди в Laravel
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Очереди запускают тяжёлые задачи (job) в фоне, чтобы не тормозить HTTP.

## Ответ

Очереди (Queues) в Laravel позволяют откладывать выполнение ресурсоёмких задач -- отправку email, генерацию PDF, обработку изображений, запросы к внешним API -- чтобы не блокировать HTTP-ответ пользователю. Вместо синхронного выполнения задача (Job) помещается в очередь, а фоновый процесс (worker) забирает и обрабатывает её асинхронно.

Laravel поддерживает несколько драйверов очередей: `sync` (для разработки, выполняет сразу), `database`, `redis`, `sqs` (Amazon), `beanstalkd`. Для продакшена рекомендуется Redis или SQS за счёт скорости и надёжности. Настройки находятся в `config/queue.php`, а текущий драйвер задаётся переменной `QUEUE_CONNECTION` в `.env`.

### Создание и структура Job

```bash
php artisan make:job SendEmailJob
```

```php
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class SendEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Максимальное число попыток
    public int $tries = 3;

    // Таймаут выполнения (секунды)
    public int $timeout = 120;

    // Backoff между попытками (секунды)
    public int $backoff = 60;

    // Уникальность job (предотвращает дубли)
    public int $uniqueFor = 3600;

    public function __construct(
        public User $user,
        public string $message
    ) {}

    /**
     * Основная логика. Зависимости можно type-hint'ить -- контейнер их зарезолвит.
     */
    public function handle(MailService $mailService): void
    {
        $mailService->send($this->user->email, $this->message);
    }

    /**
     * Определяем уникальный ID для предотвращения дублирования.
     */
    public function uniqueId(): string
    {
        return $this->user->id;
    }

    /**
     * Вызывается после исчерпания всех попыток.
     */
    public function failed(Throwable $e): void
    {
        Log::error('Email failed', [
            'user_id' => $this->user->id,
            'error'   => $e->getMessage(),
        ]);

        // Уведомить администратора
        Notification::route('slack', config('services.slack.webhook'))
            ->notify(new JobFailedNotification($this, $e));
    }
}
```

### Dispatch -- отправка в очередь

```php
// Базовый dispatch
SendEmailJob::dispatch($user, 'Welcome');

// С задержкой
SendEmailJob::dispatch($user, 'Welcome')
    ->delay(now()->addMinutes(10));

// На конкретную очередь и connection
SendEmailJob::dispatch($user, 'Welcome')
    ->onQueue('emails')
    ->onConnection('redis');

// Условный dispatch
SendEmailJob::dispatchIf($user->wantsEmail, $user, 'Welcome');

// Dispatch после ответа (после отправки HTTP response)
SendEmailJob::dispatchAfterResponse($user, 'Welcome');
```

### Job Batching (пакетная обработка)

Laravel поддерживает batch -- группу jobs, которые можно отслеживать как единое целое:

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

Bus::batch([
    new ProcessPodcast(1),
    new ProcessPodcast(2),
    new ProcessPodcast(3),
])
->then(function (Batch $batch) {
    Log::info('Все подкасты обработаны!');
})
->catch(function (Batch $batch, Throwable $e) {
    Log::error('Ошибка в batch', ['error' => $e->getMessage()]);
})
->finally(function (Batch $batch) {
    // Выполнится в любом случае
})
->allowFailures()
->dispatch();
```

### Job Chains (цепочки)

Если jobs должны выполняться последовательно:

```php
Bus::chain([
    new ProcessPayment($order),
    new SendReceipt($order),
    new UpdateInventory($order),
])->dispatch();
```

### Настройка и worker

```bash
# Для database-драйвера: создать таблицу
php artisan queue:table && php artisan migrate

# Для batch: создать таблицу
php artisan queue:batches-table && php artisan migrate

# Запуск worker (dev)
php artisan queue:work --queue=high,default --timeout=60 --tries=3

# Запуск queue:listen (перезагружается при изменениях кода, медленнее)
php artisan queue:listen

# Обработка одной задачи
php artisan queue:work --once

# Перезапуск всех workers (после деплоя!)
php artisan queue:restart
```

### Supervisor (продакшен)

В продакшене worker должен быть управляемым процессом. Supervisor автоматически перезапускает worker при падении:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/project/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/project/storage/logs/worker.log
stopwaitsecs=3600
```

### Мониторинг failed jobs

```bash
# Создать таблицу для failed jobs
php artisan queue:failed-table && php artisan migrate

# Посмотреть упавшие задачи
php artisan queue:failed

# Повторить конкретную задачу
php artisan queue:retry <id>

# Повторить все
php artisan queue:retry all

# Очистить
php artisan queue:flush
```

### Практические советы

Храните в Job только ID моделей или минимальные данные -- trait `SerializesModels` сохраняет только идентификатор и заново загружает модель из БД при обработке. Не передавайте в конструктор большие массивы или Eloquent Collections. Используйте отдельные очереди (`--queue=high,default,low`) для приоритизации задач. Всегда настраивайте `$tries`, `$timeout` и `failed()` метод. После деплоя обязательно выполняйте `php artisan queue:restart`, иначе worker будет работать со старым кодом. Для Laravel 11 рекомендуется использовать `php artisan queue:work` с флагом `--max-time=3600` вместо бесконечной работы.

## Примеры

1. `dispatch(new SendEmailJob($user))`.
2. Воркер: `php artisan queue:work`.
3. Failed jobs: `php artisan queue:failed`.

## Доп. теория

1. Драйверы: database, redis, sqs и др.
2. Ретраи и таймауты важны для стабильности.
