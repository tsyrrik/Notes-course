# Доступ к приватному свойству через Reflection
```php
class VendorClass { private $secret = 'top_secret'; }
$ref = new ReflectionClass(VendorClass::class);
$prop = $ref->getProperty('secret');
$prop->setAccessible(True);
$value = $prop->getValue(new VendorClass()); // 'top_secret'
```
Reflection позволяет изучать/менять приватные элементы (используйте осторожно).
