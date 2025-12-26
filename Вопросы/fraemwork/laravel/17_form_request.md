## Вопрос: Form Request
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Form Request выносит правила валидации/авторизации в отдельный класс.

## Ответ
Отдельный класс для валидации/авторизации.
```bash
php artisan make:request UserRequest
```
```php
class UserRequest extends FormRequest {
    public function authorize() { return true; }
    public function rules() {
        return [
            'name' => 'required|string|max:255',
            'email' => ['required','email',Rule::unique('users')->ignore($this->user)],
            'password' => 'required|min:8|confirmed',
        ];
    }
    public function messages() { return ['name.required' => 'Имя обязательно']; }
    protected function prepareForValidation() { $this->merge(['slug' => Str::slug($this->name)]); }
}
// Контроллер
public function store(UserRequest $request) { User::create($request->validated()); }
```
