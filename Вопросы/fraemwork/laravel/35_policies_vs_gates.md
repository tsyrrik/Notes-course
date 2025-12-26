## Вопрос: Policies vs Gates
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Gates — разовая проверка, Policies — набор правил для модели.

## Ответ
- Gate — одноразовая проверка через функцию/closure (может, нельзя).
- Policy — класс для модели с методами (view/update/delete и др.), структурирует правила.
```php
Gate::define('update-post', fn(User $user, Post $post) => $user->id === $post->user_id);

class PostPolicy {
    public function update(User $user, Post $post) { return $user->id === $post->user_id; }
}
```
