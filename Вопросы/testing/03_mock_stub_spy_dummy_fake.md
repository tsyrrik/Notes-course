# Mock, Stub, Spy, Dummy, Fake
| Тип | Возвр. значения | Проверяет вызовы | Настоящее поведение | Пример |
| --- | --- | --- | --- | --- |
| Dummy | ❌ | ❌ | ❌ | Пустышка для сигнатуры |
| Stub | ✅ | ❌ | ❌ | `findAll()` возвращает фиксированный список |
| Spy | ⚠️ | ✅ | ❌ | Логирует вызовы для проверки после |
| Mock | ✅ | ✅ | ❌ | Ожидания заданы заранее |
| Fake | ✅ | ❌ | ✅ (упрощ.) | In-memory репозиторий |
- Stub vs Mock: mock проверяет как вызывали. Spy vs Mock: spy проверяем после, mock — ожидания до выполнения.

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
```
