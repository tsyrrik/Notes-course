# Тестовая пирамида и AAA
- Пирамида: юнит (~70%), интеграционные (~20%), E2E/UI (~10%).
- Другие типы: функциональные, приемочные, регрессионные.
- Шаблон Arrange–Act–Assert: подготовка → действие → проверка.
- Простыми словами: сначала готовим данные/окружение (Arrange), потом выполняем одно действие (Act), потом проверяем результат (Assert). Один тест — один сценарий/действие.

## Простыми словами
- Больше быстрых юнитов, меньше тяжёлых E2E; каждый тест: подготовил → сделал действие → проверил.
```php
public function test_user_full_name(): void {
    // Arrange
    $user = new User('Иван', 'Петров');
    // Act
    $full = $user->getFullName();
    // Assert
    $this->assertSame('Иван Петров', $full);
}
```

```php
// Интеграционный: сервис + БД (in-memory)
$repo = new InMemoryUserRepository();
$service = new UserService($repo);
$service->create('a@b.c');
$this->assertNotNull($repo->findByEmail('a@b.c'));
```
