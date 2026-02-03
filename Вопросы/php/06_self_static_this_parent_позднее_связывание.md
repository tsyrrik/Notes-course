## Вопрос: self / static / $this / parent
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
`self` фиксируется на класс определения; `static` учитывает позднее статическое связывание (класс-вызвавший); `$this` — конкретный объект; `parent` — обращение к родителю.

## Ответ
- `self` — текущий класс в месте определения (без позднего связывания).
- `static` — позднее статическое связывание: класс, из которого вызвали в рантайме.
- `$this` — текущий объект (в нестатическом контексте).
- `parent` — обращение к родительскому классу.

### Когда что использовать

- `self::` — когда нужно зафиксировать поведение в базовом классе.
- `static::` — когда нужно, чтобы наследники подменяли реализацию (fluent‑интерфейсы, фабрики).
- `parent::` — когда вы переопределили метод и хотите вызвать реализацию родителя.

## Примеры
```php
class A {
    public static function who() { echo __CLASS__; }
    public static function test() { self::who(); }
    public static function testStatic() { static::who(); }
}

class B extends A {
    public static function who() { echo __CLASS__; }
}

B::test();       // A
B::testStatic(); // B
```

```php
class Base {
    public function save(): static { return $this; }
}

class Child extends Base {}

$c = new Child();
var_dump($c->save() instanceof Child); // true (static)
```

## Доп. теория
- `self::` и `__CLASS__` всегда резолвятся к классу, где метод определён; `static::` — к классу вызова.
- В статическом контексте `$this` недоступен.
