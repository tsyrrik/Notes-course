## Вопрос: Mock, Stub, Spy, Dummy, Fake

## Простой ответ
- Dummy — заглушка ради сигнатуры. Stub — вернуть заранее заданное. Mock — проверяет ожидаемые вызовы. Spy — записывает вызовы для проверки после. Fake — простая рабочая реализация без внешних зависимостей.

```php
// Stub
$mailer = $this->createStub(Mailer::class);
$mailer->method('send')->willReturn(true);

// Mock с ожиданием
$logger = $this->createMock(Logger::class);
$logger->expects($this->once())->method('error')->with('DB down');

// Fake (in-memory repo)
class InMemoryUserRepo implements UserRepository {
    private array $items = [];
    public function save(User $u): void { $this->items[$u->id] = $u; }
    public function find(int $id): ?User { return $this->items[$id] ?? null; }
}

// Spy (пример: собираем вызовы)
class SpyMailer implements Mailer {
    public array $sent = [];
    public function send(string $to, string $msg): bool {
        $this->sent[] = [$to, $msg];
        return true;
    }
}
$spy = new SpyMailer();
$service = new Notifier($spy);
$service->notify('a@b.c','Hi');
$this->assertCount(1, $spy->sent);
$this->assertSame(['a@b.c','Hi'], $spy->sent[0]);
```

## Ответ
| Тип | Возвр. значения | Проверяет вызовы | Настоящее поведение | Пример |
| --- | --- | --- | --- | --- |
| Dummy | ❌ | ❌ | ❌ | Пустышка для сигнатуры |
| Stub | ✅ | ❌ | ❌ | `findAll()` возвращает фиксированный список |
| Spy | ⚠️ | ✅ | ❌ | Логирует вызовы для проверки после |
| Mock | ✅ | ✅ | ❌ | Ожидания заданы заранее |
| Fake | ✅ | ❌ | ✅ (упрощ.) | In-memory репозиторий |
- Stub vs Mock: mock проверяет как вызывали. Spy vs Mock: spy проверяем после, mock — ожидания до выполнения.

Простыми словами и когда что брать:
- Dummy — просто заткнуть параметр, не участвует в логике (когда аргумент обязателен).
- Stub — вернуть нужные данные, не проверяя, как вызывали (когда тестируем поведение от возвращаемых значений).
- Mock — задать ожидания вызовов/аргументов заранее (когда важно, что и сколько раз вызвали).
- Spy — записать вызовы, проверить после (когда важно, как вызывали, но удобнее проверять постфактум).
- Fake — упрощённая рабочая реализация (in-memory БД/HTTP-клиент), когда нужен реальный эффект без внешних зависимостей.

## Примеры

1. Stub: репозиторий возвращает фиксированный список пользователей.
2. Mock: ожидаем, что `Mailer::send()` вызван один раз.
3. Fake: in-memory репозиторий вместо настоящей БД.

## Доп. теория

1. Mock и Spy чаще нужны в интеграционных тестах, когда важно поведение взаимодействий.
2. Чем больше моков, тем более хрупкими становятся тесты при рефакторинге.
