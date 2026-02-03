## Вопрос: Архитектура Laravel (MVC)
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
MVC делит код на модели (данные), контроллеры (логика) и представления (шаблоны), чтобы не смешивать обязанности.

## Ответ

MVC (Model-View-Controller) — это архитектурный паттерн, который разделяет приложение на три слоя с чётко определёнными обязанностями. Laravel следует этому паттерну, но дополняет его собственными абстракциями: Service Container, Service Providers, Middleware, Events и другими слоями.

**Model (Модель)** — отвечает за данные и бизнес-логику. В Laravel модели реализованы через Eloquent ORM и представляют таблицы базы данных. Каждая модель инкапсулирует правила работы с данными: связи, скоупы, аксессоры, мутаторы, касты, события жизненного цикла (creating, created, updating и т.д.). Модели располагаются в директории `app/Models/`.

**View (Представление)** — отвечает за отображение данных пользователю. В Laravel используется шаблонизатор Blade, файлы которого находятся в `resources/views/`. Blade поддерживает наследование шаблонов, компоненты, директивы и автоматическое экранирование вывода. Представления не содержат бизнес-логику — только логику отображения.

**Controller (Контроллер)** — принимает входящий HTTP-запрос, взаимодействует с моделями и возвращает ответ (представление, JSON, redirect). Контроллеры находятся в `app/Http/Controllers/`. Важно не перегружать контроллеры бизнес-логикой — для этого используют сервисные классы, Action-классы или репозитории.

```php
// Model — app/Models/User.php
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];
    protected $hidden = ['password'];

    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}

// Controller — app/Http/Controllers/UserController.php
class UserController extends Controller
{
    public function __construct(
        private UserService $userService // DI через конструктор
    ) {}

    public function index()
    {
        $users = $this->userService->getActiveUsers();
        return view('users.index', compact('users'));
    }

    public function store(UserRequest $request)
    {
        $user = $this->userService->create($request->validated());
        return redirect()->route('users.show', $user);
    }
}

// View — resources/views/users/index.blade.php
@extends('layouts.app')

@section('content')
    <h1>Пользователи</h1>
    @foreach($users as $user)
        <div class="card">
            <h2>{{ $user->name }}</h2>
            <p>{{ $user->email }}</p>
            <a href="{{ route('users.show', $user) }}">Подробнее</a>
        </div>
    @empty
        <p>Нет пользователей</p>
    @endforeach
@endsection
```

**Жизненный цикл запроса в Laravel:** HTTP-запрос проходит через `public/index.php` -> bootstrap -> Service Container -> Router -> Middleware -> Controller -> Response. Контроллер — лишь одна из точек в этой цепочке.

**Расширение MVC в реальных проектах.** На практике чистый MVC недостаточен для крупных приложений. Laravel-сообщество использует дополнительные слои:
- **Service Layer** (`app/Services/`) — бизнес-логика, вынесенная из контроллеров
- **Repository Layer** (`app/Repositories/`) — абстракция над доступом к данным
- **Action Classes** (`app/Actions/`) — отдельный класс на каждое действие (Single Responsibility)
- **DTO (Data Transfer Objects)** — типизированные объекты для передачи данных между слоями
- **Form Request** — валидация, вынесенная из контроллера

**Практический совет:** в Laravel 11 структура проекта стала минималистичнее, но принцип MVC остаётся тем же. Старайтесь держать контроллеры «тонкими» (thin controllers) — не более 5-7 строк в каждом методе. Всю бизнес-логику выносите в сервисные классы или Action-классы.

## Примеры

1. Поток: Request → Route → Controller → Response.
2. Контроллер вызывает сервис, сервис работает с моделями.
3. View рендерится через Blade и возвращается как Response.

## Доп. теория

1. Middleware формирует пайплайн до контроллера.
2. Service Container создаёт зависимости по type‑hint.
