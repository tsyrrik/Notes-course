## Вопрос: Blade: основы
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Blade — простой шаблонизатор с наследованием, компонентами и автоматическим экранированием.

## Ответ
- Шаблонизатор с экранированием/вставками, условиями и циклами.
```blade
<h1>{{ $title }}</h1>
<p>{!! $htmlContent !!}</p> {{-- не экранируется --}}

@if($user->isAdmin()) Админ @else Пользователь @endif

@foreach($users as $user)
  <div>{{ $user->name }}</div>
@empty
  <p>Нет пользователей</p>
@endforeach
```

## Наследование
```blade
{{-- layouts/app.blade.php --}}
<title>@yield('title','Default')</title>
<body>@yield('content')</body>

{{-- pages/home.blade.php --}}
@extends('layouts.app')
@section('title','Главная')
@section('content') <h1>Добро пожаловать!</h1> @endsection
```

## Компоненты
```bash
php artisan make:component Alert
```
```blade
<x-alert type="success" message="Сохранено!" />
<x-button class="btn-primary">Нажми</x-button>
```

## Примеры

1. `@extends('layout')` и `@section('content')`.
2. `@foreach($users as $user)`.
3. Компоненты: `<x-alert type="success"/>`.

## Доп. теория

1. Blade кэшируется в `storage/framework/views`.
2. Вывод `{{ }}` экранируется по умолчанию.
