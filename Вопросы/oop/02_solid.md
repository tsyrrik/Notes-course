# SOLID
- S (Single Responsibility): одна причина для изменения.
- O (Open/Closed): открыто для расширения, закрыто для модификации.
- L (Liskov Substitution): наследник не ужесточает предусловия, не ослабляет постусловия; инварианты сохраняются.
- I (Interface Segregation): много маленьких интерфейсов лучше одного «толстого».
- D (Dependency Inversion): зависеть от абстракций, а не реализаций. Injections: constructor (желательно), setter, property (нежелательно).

```php
// S: отдельный класс пишет лог, другой — считает
class Logger { public function info(string $m): void {/*...*/} }
class ReportBuilder { public function __construct(private Logger $log) {} }

// O: добавление нового уведомления без правки существующего кода
interface Notifier { public function send(string $msg): void; }
class SmsNotifier implements Notifier { /* ... */ }
class EmailNotifier implements Notifier { /* ... */ }

// L: наследник не сужает контракт базового типа
interface Bird { public function fly(): void; }
class Sparrow implements Bird { public function fly(): void {/*...*/} }

// I: разбиваем неподъемный интерфейс
interface Printer { public function print(string $doc): void; }
interface Scanner { public function scan(): string; }

// D: зависим от абстракции, внедряем через конструктор
class UserService {
    public function __construct(private Notifier $notifier) {}
    public function welcome(): void { $this->notifier->send('Hi'); }
}
```
