# Конфликт методов трейта и класса
Метод класса имеет приоритет над методом трейта. Чтобы вызвать метод трейта, создайте алиас через `insteadof`/`as`.
```php
trait Hello { public function sayHello(){ echo 'Trait'; } }
class Hi { use Hello { sayHello as traitHello; }
  public function sayHello(){ echo 'Class then '; $this->traitHello(); }
}
```
