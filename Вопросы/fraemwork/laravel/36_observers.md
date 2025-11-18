# Model Observers

Простыми словами: Observer ловит события модели (creating/updated) и выносит логику из модели.
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
