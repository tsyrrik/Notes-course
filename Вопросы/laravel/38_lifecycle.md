# Жизненный цикл HTTP-запроса в Laravel

Простыми словами: Путь HTTP-запроса через Kernel, middleware, роутер, контроллер, отклик.
1. Запрос → `public/index.php`.
2. Инициализация приложения и `App\Http\Kernel`.
3. Kernel строит pipeline глобальных middleware.
4. Router подбирает маршрут, применяет route-middleware, binding.
5. Вызывается контроллер/closure, бизнес-логика, сервисы, модели, события.
6. Возвращается `Response` (view/JSON/redirect/stream).
7. Terminable middleware (логи/метрики).
8. Kernel отправляет HTTP-ответ клиенту.
