# Blade: основы

Простыми словами: Blade — простой шаблонизатор с наследованием, компонентами и автоматическим экранированием.
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
