## Вопрос: Model Observers
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Observer ловит события модели (creating/updated) и выносит логику из модели.

## Ответ
Переносят обработку событий модели в отдельный класс.
```bash
php artisan make:observer UserObserver --model=User
```
```php
class UserObserver {
    public function creating(User $user) { $user->uuid = Str::uuid(); }
    public function created(User $user) { event(new UserRegistered($user)); }
}
// Регистрация (AppServiceProvider boot)
User::observe(UserObserver::class);
```

## Примеры

1. `php artisan make:observer UserObserver --model=User`.
2. Регистрация в `AppServiceProvider`.
3. Хук `created()` для аудита.

## Доп. теория

1. Observers слушают события моделей (created/updated).
2. Тяжёлые задачи лучше отправлять в очередь.
