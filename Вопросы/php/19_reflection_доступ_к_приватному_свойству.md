# Доступ к приватному свойству через Reflection
Простыми словами: отражение (Reflection) позволяет заглядывать и менять приватные элементы — полезно в тестах/инструментах, но ломает инкапсуляцию.
```php
class VendorClass { private $secret = 'top_secret'; }
$ref = new ReflectionClass(VendorClass::class);
$prop = $ref->getProperty('secret');
$prop->setAccessible(True);
$value = $prop->getValue(new VendorClass()); // 'top_secret'
```
Reflection позволяет изучать/менять приватные элементы (используйте осторожно — только в тестах/инфраструктуре, не в боевой логике).
