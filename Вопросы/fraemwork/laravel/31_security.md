## Вопрос: Безопасность в Laravel
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Встроенные защиты — CSRF, экранирование XSS, параметризованные запросы, авторизация.

## Ответ
- CSRF защита (middleware), `@csrf` в формах; заголовок `X-CSRF-TOKEN` в AJAX.
- SQL injection: параметризованные запросы, автоматическое экранирование.
- XSS: Blade экранирует `{{ }}`, доверенный HTML — через `{!! !!}`.
- Авторизация: Policies/Gates, middleware `auth`/`can`.
- Валидация входа: правила длины/форматов, mime для файлов.

## Примеры

1. Форма: `@csrf` + middleware `VerifyCsrfToken`.
2. Авторизация: `Gate::define('update-post', ...)` и `$this->authorize('update', $post)`.
3. Валидация файлов: `['avatar' => 'image|max:2048']`.

## Доп. теория

1. Mass assignment контролируется через `$fillable`/`$guarded`.
2. Для SPA используйте CSRF‑cookie + `auth:sanctum`.
