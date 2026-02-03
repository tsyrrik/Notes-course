# Вопросы по фреймворкам (Laravel и Symfony)

## Простой ответ
Eloquent (Active Record) — модель сама сохраняет себя (`save/delete`). Doctrine (Data Mapper) — сущность «не знает» про БД, сохраняет `EntityManager`.

## Ответ

### Doctrine (Data Mapper) vs Active Record (Eloquent)

| Аспект | Active Record (Eloquent) | Data Mapper (Doctrine) |
| --- | --- | --- |
| Модель | Знает про БД, сама `save()`/`delete()` | Entity — чистый PHP-объект, без знания о БД |
| Сохранение | `$user->save()` | `$em->persist($user); $em->flush()` |
| Связь с таблицей | 1 модель = 1 таблица | Маппинг через атрибуты/XML/YAML |
| DDD | Сложно, модель перегружена | Идеально — entity = domain object |
| Тестирование | Нужен БД или mock модели | Entity тестируется как обычный PHP-класс |
| Порог входа | Низкий | Выше |
| Производительность | Каждый `save()` — запрос | Unit of Work батчит запросы при `flush()` |

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

## Плюсы/минусы Laravel и Symfony

### Laravel

**Плюсы:**
- Быстрый старт, много готовых решений (Eloquent, Blade, Queue, Broadcast, Sanctum)
- DX (Developer Experience): Artisan, Sail, Telescope, Horizon, Valet
- Большая экосистема пакетов и сообщество
- Отличная документация с примерами
- Rapid prototyping — MVP за часы

**Минусы:**
- Много «магии» (Facades, auto-discovery) — сложнее дебажить
- Active Record (Eloquent) плохо масштабируется для сложного домена
- Частые мажорные релизы (ежегодно)
- Связность с фреймворком — сложно мигрировать

### Symfony

**Плюсы:**
- Модульность — можно использовать только нужные компоненты
- Строгая типизация, стандарты (PSR), прозрачность
- Data Mapper (Doctrine) — идеально для DDD и сложного домена
- Гибкий DI/конфиг, компиляция контейнера
- Стабильные LTS (4 года поддержки), предсказуемые апгрейды
- Компоненты Symfony используются в Laravel, Drupal, Magento и др.

**Минусы:**
- Выше порог входа
- Больше конфигов и boilerplate в начале
- Меньше «из коробки» для быстрых MVP
- Документация более академичная

### Когда что выбирать

| Сценарий | Рекомендация |
| --- | --- |
| MVP, стартап, CRUD-приложение | Laravel |
| Сложный домен, enterprise | Symfony |
| Микросервисы | Symfony (компоненты) или Laravel (Lumen/Octane) |
| API-first проект | Оба хорошо (Laravel Sanctum / Symfony API Platform) |
| Команда из джунов | Laravel (проще старт) |
| Долгосрочный проект с DDD | Symfony + Doctrine |

## Примеры

1. Пример базового использования: Вопросы по фреймворкам (Laravel и Symfony).
2. Пример типичной задачи/сценария с Вопросы по фреймворкам (Laravel и Symfony).

## Доп. теория

1. Ключевые термины и зависимости вокруг темы: Вопросы по фреймворкам (Laravel и Symfony).
2. Типичные ошибки и ограничения при работе с Вопросы по фреймворкам (Laravel и Symfony).

