## Вопрос: Form Request
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Form Request выносит правила валидации/авторизации в отдельный класс.

## Ответ

Form Request — это специальный класс, который инкапсулирует правила валидации и авторизации запроса. Вместо того чтобы загромождать контроллер правилами валидации, вы выносите их в отдельный класс, который Laravel автоматически применяет до вызова метода контроллера. Это следование принципу Single Responsibility — контроллер обрабатывает, Form Request валидирует.

**Как это работает:** когда Laravel видит type-hint `StoreUserRequest` в сигнатуре метода контроллера, он автоматически создаёт экземпляр, вызывает `authorize()`, затем `rules()`, и если валидация провалилась — перенаправляет назад (веб) или возвращает 422 (API). Контроллер вызывается только при успешной валидации.

```bash
php artisan make:request StoreUserRequest
php artisan make:request UpdateUserRequest
```

**Полный пример Form Request:**

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    // Авторизация — может ли пользователь выполнить этот запрос
    public function authorize(): bool
    {
        // true — разрешить всем (аутентификация через middleware)
        // или проверка через Policy:
        // return $this->user()->can('create', User::class);
        return true;
    }

    // Правила валидации
    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'max:255'],
            'email'    => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'confirmed', Password::min(8)->mixedCase()->numbers()],
            'role'     => ['required', Rule::in(['user', 'editor', 'admin'])],
            'avatar'   => ['nullable', 'image', 'max:2048'],
            'tags'     => ['nullable', 'array'],
            'tags.*'   => ['string', 'max:50'],
        ];
    }

    // Кастомные сообщения
    public function messages(): array
    {
        return [
            'name.required'     => 'Укажите имя пользователя',
            'email.unique'      => 'Пользователь с таким email уже существует',
            'password.confirmed' => 'Пароли не совпадают',
        ];
    }

    // Кастомные имена атрибутов (для автогенерации сообщений)
    public function attributes(): array
    {
        return [
            'name'  => 'имя',
            'email' => 'электронная почта',
        ];
    }

    // Подготовка данных ДО валидации
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug'  => Str::slug($this->name),
            'email' => Str::lower($this->email),
        ]);
    }

    // Обработка ПОСЛЕ успешной валидации
    protected function passedValidation(): void
    {
        // Например, убрать лишние пробелы
        $this->replace([
            ...$this->validated(),
            'name' => trim($this->name),
        ]);
    }
}
```

**Отдельный Form Request для update (с учётом текущей записи):**

```php
class UpdateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        // $this->route('user') — модель из Route Model Binding
        return $this->user()->can('update', $this->route('user'));
    }

    public function rules(): array
    {
        $userId = $this->route('user')->id; // текущий пользователь

        return [
            'name'  => ['required', 'string', 'max:255'],
            // unique, но игнорировать текущую запись
            'email' => ['required', 'email', Rule::unique('users')->ignore($userId)],
            // пароль необязателен при обновлении
            'password' => ['nullable', 'confirmed', Password::min(8)],
        ];
    }
}
```

**Условная валидация:**

```php
class OrderRequest extends FormRequest
{
    public function rules(): array
    {
        $rules = [
            'type'    => ['required', Rule::in(['delivery', 'pickup'])],
            'items'   => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity'   => ['required', 'integer', 'min:1'],
        ];

        // Условные правила через when/sometimes
        if ($this->type === 'delivery') {
            $rules['address'] = ['required', 'string', 'max:500'];
            $rules['city']    = ['required', 'string'];
        }

        return $rules;
    }

    // Альтернатива — метод withValidator
    public function withValidator($validator): void
    {
        $validator->sometimes('coupon', 'exists:coupons,code', function ($input) {
            return $input->has('coupon') && $input->coupon !== '';
        });
    }
}
```

**Использование в контроллере:**

```php
class UserController extends Controller
{
    public function store(StoreUserRequest $request)
    {
        // Валидация уже пройдена — работаем с чистыми данными
        $user = User::create($request->validated());
        return redirect()->route('users.show', $user);
    }

    public function update(UpdateUserRequest $request, User $user)
    {
        // safe() возвращает ValidatedInput — альтернатива validated()
        $user->update($request->safe()->except(['password']));

        if ($request->filled('password')) {
            $user->update(['password' => $request->password]);
        }

        return redirect()->route('users.show', $user);
    }
}
```

**Практический совет:** создавайте отдельные Form Request для store и update — правила часто отличаются (например, пароль обязателен при создании, но не при обновлении). Используйте `$request->validated()` или `$request->safe()` — они возвращают только проверенные данные, что защищает от mass assignment. Метод `authorize()` — удобное место для проверки Policy. Если `authorize()` возвращает `false`, Laravel автоматически возвращает 403. Метод `prepareForValidation()` полезен для нормализации данных (trim, slug, lowercase) до проверки.

## Примеры

1. `php artisan make:request StorePostRequest`.
2. `rules()` + `authorize()`.
3. Автоматическая валидация при инъекции в контроллер.

## Доп. теория

1. FormRequest инкапсулирует валидацию и авторизацию.
2. Метод `authorize()` решает доступ к действию.
