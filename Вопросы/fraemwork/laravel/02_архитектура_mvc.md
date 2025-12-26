## Вопрос: Архитектура Laravel (MVC)
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
MVC делит код на модели (данные), контроллеры (логика) и представления (шаблоны), чтобы не смешивать обязанности.

## Ответ
- Model: Eloquent модели для работы с БД.
- View: Blade шаблоны.
- Controller: обработка запросов и бизнес-логика.
```php
// Model
class User extends Model { protected $fillable = ['name','email']; }

// Controller
class UserController extends Controller {
    public function index() {
        $users = User::all();
        return view('users.index', compact('users'));
    }
}

// View resources/views/users/index.blade.php
@foreach($users as $user)
    <p>{{ $user->name }}</p>
@endforeach
```
