## Вопрос: API Resources
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
API Resource превращает модель в аккуратный JSON с нужными полями/условиями.

## Ответ

API Resources в Laravel -- это трансформационный слой между моделями Eloquent и JSON-ответами API. Они решают задачу контроля над тем, какие данные и в каком формате отдаются клиенту. Без resources вы вынуждены вручную формировать массивы или возвращать модель "как есть", рискуя отдать лишние поля (password_hash, внутренние ID, служебные поля).

Resource -- это класс, который принимает модель (или коллекцию) и описывает, как преобразовать её в массив для JSON. Laravel автоматически сериализует этот массив в JSON при возврате из контроллера. Resources поддерживают условные поля, вложенные resources, пагинацию и meta-данные.

### Создание Resource

```bash
php artisan make:resource UserResource
php artisan make:resource UserCollection   # для коллекций
```

### Базовый Resource

```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'avatar_url' => $this->avatar_url,
            'created_at' => $this->created_at->toDateTimeString(),

            // Условное поле -- включается только если связь загружена
            'posts'       => PostResource::collection($this->whenLoaded('posts')),
            'posts_count' => $this->whenCounted('posts'),

            // Условное поле -- включается по условию
            'is_admin'    => $this->when($request->user()?->isAdmin(), $this->is_admin),

            // Поле только для конкретного пользователя (скрыть от других)
            'email_verified_at' => $this->when(
                $request->user()?->id === $this->id,
                $this->email_verified_at?->toDateTimeString()
            ),

            // Вложенный resource
            'profile' => new ProfileResource($this->whenLoaded('profile')),

            // Merge условный блок полей
            $this->mergeWhen($request->user()?->isAdmin(), [
                'internal_notes' => $this->internal_notes,
                'last_login_ip'  => $this->last_login_ip,
            ]),
        ];
    }
}
```

### Использование в контроллере

```php
class UserController extends Controller
{
    // Одна модель
    public function show(User $user): UserResource
    {
        $user->load(['profile', 'posts']);
        return new UserResource($user);
    }

    // Коллекция
    public function index(): AnonymousResourceCollection
    {
        $users = User::with('profile')
            ->withCount('posts')
            ->paginate(15);

        return UserResource::collection($users);
        // Автоматически добавляет пагинацию (links, meta)
    }
}
```

### Resource Collection (кастомная коллекция)

Для добавления meta-данных или обёртки коллекции:

```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    // Указываем, какой Resource использовать для каждого элемента
    public $collects = UserResource::class;

    public function toArray($request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total_admins' => $this->collection->where('is_admin', true)->count(),
            ],
        ];
    }

    // Дополнительные meta-данные в корень ответа
    public function with($request): array
    {
        return [
            'api_version' => '1.0',
            'generated_at' => now()->toISOString(),
        ];
    }
}

// В контроллере:
return new UserCollection(User::paginate(15));
```

### Response codes и headers

```php
// В контроллере -- кастомный статус и заголовки
return (new UserResource($user))
    ->response()
    ->setStatusCode(201)
    ->header('X-Custom-Header', 'value');
```

### Wrapping (обёртка data)

По умолчанию resource оборачивает ответ в ключ `data`. Это можно отключить:

```php
// В AppServiceProvider::boot()
JsonResource::withoutWrapping();

// Результат без обёртки:
// { "id": 1, "name": "John" }
// Вместо:
// { "data": { "id": 1, "name": "John" } }
```

### Практические советы

Всегда используйте API Resources вместо прямого возврата моделей -- это защитит от утечки данных и даст контроль над форматом API. Используйте `whenLoaded()` для связей, чтобы избежать N+1 и не отдавать null-поля для незагруженных связей. Для пагинированных результатов `UserResource::collection($paginator)` автоматически добавит meta с информацией о страницах. Создавайте разные Resources для разных контекстов (UserBriefResource для списков, UserDetailResource для детальной страницы). В Laravel 11 resources остаются рекомендованным подходом для REST API.

## Примеры

1. `UserResource::collection($users)`.
2. `return new UserResource($user)`.
3. `with()` для добавления метаданных.

## Доп. теория

1. Resources — слой трансформации, не бизнес‑логика.
2. Пагинация автоматически форматируется.
