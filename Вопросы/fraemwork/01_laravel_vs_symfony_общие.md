# Вопросы по фреймворкам (Laravel и Symfony)

## Простой ответ
Eloquent (Active Record) — модель сама сохраняет себя (`save/delete`). Doctrine (Data Mapper) — сущность «не знает» про БД, сохраняет `EntityManager`.

## Ответ
## Doctrine (Data Mapper) vs Active Record (Eloquent)

- Active Record: логика сохранения внутри модели; проще старт, жёстче связка с БД.
- Data Mapper: Entity + Unit of Work/EntityManager; гибче, чище для DDD.

## Eloquent ORM (Laravel)
Простыми словами: таблица = класс, строка = объект; модель умеет `find/save/delete`, связи `hasOne/hasMany/belongsTo/ManyToMany`, lazy/eager загрузка. Запросы пишем методами, а не SQL.

## Doctrine ORM (Symfony)
Простыми словами: Entity — только данные/логика домена; `EntityManager` управляет жизненным циклом (`persist/flush`). Основано на Data Mapper, гибко для сложных доменов и DDD.

## Timestampable + Listener (Doctrine)
Простыми словами: поля `createdAt/updatedAt` обновляются автоматически в слушателе жизненного цикла (например, `prePersist`, `preUpdate`), чтобы не писать `setUpdatedAt()` руками.

## Middleware
Простыми словами: прослойка до/после контроллера — авторизация, CSRF, логирование, заголовки, локаль. Может пропустить, изменить или вернуть ответ.

## Сервис в Laravel
Простыми словами: класс с бизнес-логикой, регистрируется/внедряется через контейнер.  
```php
class PaymentService { public function pay(int $orderId) {/*...*/} }
class CheckoutController {
    public function __construct(private PaymentService $payments) {}
}
```

## Service Provider (Laravel)
Простыми словами: точка регистрации/настройки сервисов при запуске приложения.  
- `register()` — биндинги в контейнер.  
- `boot()` — логика, когда зависимости уже доступны (events, routes и т.д.).

## Service Container (Laravel/Symfony)
Простыми словами: IoC-контейнер создаёт и отдаёт зависимости. Вызываешь класс — контейнер подставляет нужные объекты (DI), проще тестировать и менять реализации.

## Жизненный цикл запроса Laravel (коротко)
`public/index.php` → `bootstrap/app.php` (приложение/контейнер) → HTTP Kernel (bootstrappers, провайдеры) → глобальные middleware → роутер → middleware маршрута → контроллер/замыкание → ответ → обратный проход через middleware → отправка клиенту (`terminate()`).

## Зачем провайдеры, где middleware, как после ответа?
- Провайдеры: регистрируют/инициализируют сервисы (главный «крюк» старта).  
- `register()` vs `boot()`: в register объявляем биндинги, в boot используем уже доступные зависимости.  
- Middleware: `app/Http/Kernel.php` — глобальные, группы (web/api), алиасы для маршрутов.  
- Post-response: `terminate()` в middleware, события/очереди.

## Плюсы/минусы Laravel и Symfony (кратко)
- Laravel плюсы: быстрый старт, много готовых решений (Eloquent, Blade, Queue, Broadcast), мощный DX (Artisan, Sail, Telescope), большая экосистема пакетов.  
- Laravel минусы: сильнее «магия» и связность с Eloquent; частые мажоры; Active Record хуже для сложного домена; потенциально «толстый» runtime.

- Symfony плюсы: модульность, строгая типизация/стандарты, Data Mapper (Doctrine) хорошо для DDD, гибкий DI/конфиг, можно собирать только нужные компоненты, стабильные долгоживущие LTS.  
- Symfony минусы: выше порог входа, больше настроек/конфигов, меньше «из коробки» для быстрых MVP (если без Flex/UX), шаблон кода чуть более многословен в старте.
