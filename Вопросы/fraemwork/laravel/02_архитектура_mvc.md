# Архитектура Laravel (MVC)

Простыми словами: MVC делит код на модели (данные), контроллеры (логика) и представления (шаблоны), чтобы не смешивать обязанности.
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
