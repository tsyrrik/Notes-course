## Вопрос: Policies vs Gates
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

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

## Примеры

1. Gate: `Gate::define('edit-post', fn($u,$p) => ...)`.
2. Policy: `PostPolicy@update`.
3. Проверка: `$this->authorize('update', $post)`.

## Доп. теория

1. Gates — простые проверки, Policies — для моделей.
2. Политики можно регистрировать автоматически.
