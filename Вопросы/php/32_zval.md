## Вопрос: Zval в PHP
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
- Одно значение хранится как zval с refcount: копии разделяют память до записи, ссылки ломают COW и связывают переменные.

## Ответ

### Что такое zval

`zval` (Zend Value) — базовая структура данных в движке PHP (Zend Engine), в которой хранится **каждое** значение. На 64-битной системе:

```c
// Упрощённая структура zval (PHP 8.x)
struct zval {
    zend_value value;    // 8 байт — union: long, double, pointer на string/array/object
    uint32_t   type_info; // 4 байта — тип + флаги (is_ref, gc_info)
    // итого: 16 байт с выравниванием
};
```

### Типы значений в zval

| Тип PHP | Хранение в zval |
| --- | --- |
| `int` | Inline в `zend_long` (8 байт) |
| `float` | Inline в `double` (8 байт) |
| `bool` | Inline как `zend_long` (0 или 1) |
| `null` | Только тег типа, значения нет |
| `string` | Указатель на `zend_string` (refcounted, immutable для interned) |
| `array` | Указатель на `HashTable` (refcounted) |
| `object` | Указатель на `zend_object` (refcounted) |

### Refcount и Copy-on-Write

```php
$a = [1, 2, 3]; // zval: array, refcount = 1
$b = $a;         // refcount = 2, данные НЕ копируются
$b[] = 4;        // refcount $a = 1, создана копия для $b (separation)
```

Скаляры (`int`, `float`, `bool`, `null`) **не используют refcount** — они копируются inline (слишком малы, чтобы COW имел смысл).

### Ссылки (references)

```php
$a = 42;
$b = &$a;  // создаётся zend_reference, оборачивающая zval
$b = 100;  // $a тоже = 100
unset($b); // разрывается ссылка, $a остаётся = 100
```

Ссылки создают дополнительную обёртку `zend_reference` и **ломают COW** — при присваивании `$c = $a` (если `$a` — ссылка) данные будут скопированы, а не разделены.

### Garbage Collector

- **Refcount** — основной механизм: когда refcount = 0, значение освобождается
- **Cycle collector** — для циклических ссылок (массив ссылается сам на себя). Срабатывает при достижении порога потенциальных циклов (по умолчанию 10 000)

```php
// Циклическая ссылка — refcount никогда не станет 0
$a = [];
$a[] = &$a; // GC очистит через cycle collector
unset($a);
gc_collect_cycles(); // принудительная очистка
```

## Примеры
```php
$a = [1, 2, 3];
$b = $a;     // COW — пока копии нет
$b[] = 4;    // separation — копия создаётся
```

### Диагностика

```php
debug_zval_refcount($var);  // показывает refcount (не точный — сам вызов влияет)
memory_get_usage();         // текущее потребление памяти
memory_get_peak_usage();    // пиковое потребление
gc_status();                // статистика GC
```

## Доп. теория
- Размеры zval/структур зависят от версии и сборки, но порядок величин остаётся стабильным.
- `debug_zval_refcount()` влияет на измерение refcount — это диагностический, а не точный инструмент.
