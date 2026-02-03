## Вопрос: SOLID

## Простой ответ
- S: один повод менять класс. O: расширяем без правки. L: наследник ведёт себя как базовый тип. I: лучше много маленьких интерфейсов. D: зависим от абстракций, внедряем зависимости.

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

## Ответ

### S — Single Responsibility Principle (Принцип единственной ответственности)

Класс должен иметь **одну причину для изменения**. Если класс отвечает за несколько вещей, изменение одной может сломать другую.

**Плохо**: `UserService` и валидирует, и отправляет email, и записывает в БД.
**Хорошо**: `UserValidator`, `UserMailer`, `UserRepository` — каждый делает своё.

### O — Open/Closed Principle (Принцип открытости/закрытости)

Класс должен быть **открыт для расширения** и **закрыт для модификации**. Добавляйте новое поведение через новые классы (наследование, реализацию интерфейсов), а не правкой существующего кода.

**Реализация**: используйте интерфейсы и Strategy/Decorator паттерны.

### L — Liskov Substitution Principle (Принцип подстановки Лисков)

Объект подкласса должен быть **заменим** на объект базового класса без нарушения корректности программы.

**Нарушение**: `class Square extends Rectangle` — при изменении ширины квадрат меняет и высоту, ломая ожидания кода, работающего с Rectangle.

**Правила**:
- Наследник не ужесточает предусловия (не требует больше)
- Наследник не ослабляет постусловия (гарантирует не меньше)
- Инварианты родителя сохраняются

### I — Interface Segregation Principle (Принцип разделения интерфейсов)

Много маленьких специализированных интерфейсов лучше одного «толстого». Клиент не должен зависеть от методов, которые не использует.

```php
// Плохо: один толстый интерфейс
interface Worker {
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

// Хорошо: разделение
interface Workable { public function work(): void; }
interface Feedable { public function eat(): void; }
```

### D — Dependency Inversion Principle (Принцип инверсии зависимостей)

Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от **абстракций**.

```php
// Плохо: зависимость от конкретного класса
class OrderService {
    private MySQLRepository $repo; // жёсткая связь
}

// Хорошо: зависимость от абстракции
class OrderService {
    public function __construct(
        private OrderRepositoryInterface $repo, // можно подменить
    ) {}
}
```

**Способы внедрения**: конструктор (предпочтительно), сеттер (опционально), свойство (нежелательно).

## Примеры
```text
SRP: класс отвечает за одну причину изменения.
OCP: новое поведение добавляем через новые классы, не правим старые.
```

## Доп. теория
- SOLID — набор эвристик, а не строгих законов; применять осмысленно.
- Чрезмерная абстракция может усложнить простой код.
