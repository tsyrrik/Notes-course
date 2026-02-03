## Вопрос: Роутинг
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
сопоставляет URL/метод с контроллером. Описывается в YAML/PHP/аннотациях/атрибутах.

## Ответ

### Принцип работы роутинга

Роутинг в Symfony -- это механизм сопоставления входящего HTTP-запроса (URL + метод) с конкретным контроллером. Компонент Routing анализирует URI, применяет шаблоны маршрутов и определяет, какой метод контроллера должен обработать запрос. В Symfony 7 рекомендуемый способ определения маршрутов -- через PHP-атрибуты `#[Route]`.

### Способы определения маршрутов

Маршруты можно описать четырьмя способами: **атрибуты PHP 8** (рекомендуется), **YAML**, **XML** и **PHP-конфиг**. Атрибуты наиболее удобны, потому что маршрут находится рядом с кодом контроллера.

```php
use Symfony\Component\Routing\Attribute\Route;

class UserController extends AbstractController
{
    // Базовый маршрут
    #[Route('/users', name: 'users_index', methods: ['GET'])]
    public function index(): Response { /* ... */ }

    // Маршрут с параметром и требованием
    #[Route('/users/{id}', name: 'user_show', requirements: ['id' => '\d+'], methods: ['GET'])]
    public function show(int $id): Response { /* ... */ }

    // Маршрут с дефолтным значением
    #[Route('/users/page/{page}', name: 'users_paginated', defaults: ['page' => 1])]
    public function paginated(int $page): Response { /* ... */ }
}
```

```yaml
# config/routes.yaml
home:
    path: /
    controller: App\Controller\HomeController::index

user_show:
    path: /users/{id}
    controller: App\Controller\UserController::show
    requirements:
        id: '\d+'
    methods: [GET]
```

### Префиксы и группировка маршрутов

Атрибут `#[Route]` на уровне класса задаёт префикс для всех маршрутов контроллера:

```php
#[Route('/api/v1/users', name: 'api_users_')]
class ApiUserController extends AbstractController
{
    #[Route('', name: 'index', methods: ['GET'])]       // /api/v1/users
    public function index(): JsonResponse { /* ... */ }

    #[Route('/{id}', name: 'show', methods: ['GET'])]   // /api/v1/users/{id}
    public function show(int $id): JsonResponse { /* ... */ }

    #[Route('', name: 'create', methods: ['POST'])]     // /api/v1/users (POST)
    public function create(): JsonResponse { /* ... */ }
}
```

### Генерация URL

Для генерации URL из имени маршрута используется `UrlGeneratorInterface` или хелпер `generateUrl()` в AbstractController:

```php
// В контроллере
$url = $this->generateUrl('user_show', ['id' => 42]);
// Результат: /users/42

// Абсолютный URL
$absoluteUrl = $this->generateUrl('user_show', ['id' => 42], UrlGeneratorInterface::ABSOLUTE_URL);
```

```twig
{# В Twig-шаблоне #}
<a href="{{ path('user_show', {id: user.id}) }}">Профиль</a>
<a href="{{ url('user_show', {id: user.id}) }}">Абсолютная ссылка</a>
```

### Специальные параметры и условия

Symfony поддерживает специальные параметры маршрута (`_format`, `_locale`), а также выражения условий через `condition`:

```php
#[Route('/articles/{slug}.{_format}', name: 'article_show',
    requirements: ['_format' => 'html|json'],
    defaults: ['_format' => 'html']
)]
public function show(string $slug, string $_format): Response { /* ... */ }

// Условие на основе запроса
#[Route('/api/data', name: 'api_data', condition: "request.headers.get('Accept') matches '/json/'")]
public function apiData(): JsonResponse { /* ... */ }
```

### Приоритет и порядок

Маршруты проверяются в порядке их определения. Если два маршрута могут совпасть с одним URL, выигрывает тот, который определён раньше. Для явного управления приоритетом используется параметр `priority`:

```php
#[Route('/users/new', name: 'user_new', priority: 1)]    // Проверяется первым
public function new(): Response { /* ... */ }

#[Route('/users/{id}', name: 'user_show')]                 // Проверяется вторым
public function show(int $id): Response { /* ... */ }
```

### Отладка маршрутов

```bash
# Показать все маршруты
php bin/console debug:router

# Проверить, какой маршрут совпадёт с URL
php bin/console router:match /users/42

# Показать детали конкретного маршрута
php bin/console debug:router user_show
```

### Практические советы

- Всегда указывайте `methods` в маршрутах для явного определения допустимых HTTP-методов
- Давайте маршрутам осмысленные имена (`name`), используя snake_case с префиксом контекста (например, `admin_user_edit`)
- Используйте `requirements` для ограничения типов параметров (например, `'\d+'` для числовых id)
- В Symfony 7 класс атрибута переименован из `Symfony\Component\Routing\Annotation\Route` в `Symfony\Component\Routing\Attribute\Route`

## Примеры

1. `#[Route('/posts/{id}', requirements: ['id' => '\d+'])]`.
2. Префикс контроллера `#[Route('/api', name: 'api_')]`.
3. Генерация URL: `path('user_show', {id: 10})` в Twig.

## Доп. теория

1. Порядок маршрутов важен, более специфичные должны идти раньше.
2. `router:match` помогает отладить конфликт правил.
