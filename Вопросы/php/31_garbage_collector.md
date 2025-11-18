# Garbage Collector (GC) в PHP
- PHP управляет памятью через счётчик ссылок; циклы уничтожает GC.
- Цикл: сбор подозрительных zval → периодический проход → уничтожение циклов.
- Настройки: `zend.enable_gc`, `gc_enable()/gc_disable()`, `gc_collect_cycles()`, `gc_status()`.
```php
$a = new stdClass(); $b = new stdClass();
$a->b = $b; $b->a = $a;
unset($a, $b);
echo gc_collect_cycles(); // 2 — собрал цикл
```
- Советы: избегать лишних циклических ссылок и замыканий на `$this`; в долгоживущих воркерах периодически вызывать `gc_collect_cycles()`.
