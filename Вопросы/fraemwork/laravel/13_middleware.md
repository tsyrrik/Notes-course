# Middleware

Простыми словами: Middleware — фильтры запросов (auth, логирование, CORS, throttle) до/после контроллера.
Прослойки для фильтрации запросов.
```bash
php artisan make:middleware CheckAge
```
```php
class CheckAge {
    public function handle($request, Closure $next, $minAge = 18) {
        if ($request->age < $minAge) {
            return redirect('home')->with('error','Недостаточный возраст');
        }
        return $next($request);
    }
    public function terminate($request, $response) { Log::info('done'); }
}

// Регистрация в Kernel
protected $middlewareAliases = ['check.age' => \App\Http\Middleware\CheckAge::class];
Route::get('/dashboard', [DashboardController::class,'index'])->middleware('check.age:21');
```

## Типы
- Global (`$middleware`), Groups (`$middlewareGroups` web/api), Route (`$middlewareAliases`), terminable.
