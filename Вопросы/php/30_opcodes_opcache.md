# OpCodes и OpCache

Простыми словами: PHP-файл парсится в AST, компилируется в opcodes (байткод), которые исполняет Zend VM. OpCache кеширует opcodes между запросами, JIT (PHP 8) может превращать “горячие” opcodes в машинный код.

- Путь: source → lexer → parser → AST → compile → opcodes → Zend VM.
- OpCache: `opcache.enable=1`, `memory_consumption`, `max_accelerated_files`, `validate_timestamps` и др. Кеширует байткод, чтобы не компилировать каждый запрос.
- JIT: оптимизирует CPU-heavy трассы; для IO-bound/API типично мало эффекта.
