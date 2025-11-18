# self / static / $this / parent
Простыми словами: `self` фиксируется на класс определения; `static` учитывает позднее статическое связывание (класс-вызвавший); `$this` — конкретный объект; `parent` — обращение к родителю.
- `self` — текущий класс в месте определения, без позднего связывания.
- `static` — позднее статическое связывание (класс, из которого вызвали).
- `$this` — текущий объект.
- `parent` — обращение к родительскому классу.
```php
class A { public static function who(){echo __CLASS__;} public static function test(){self::who();} public static function testStatic(){static::who();}}
class B extends A { public static function who(){echo __CLASS__;} }
B::test();       // A
B::testStatic(); // B
```
