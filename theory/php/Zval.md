## Что это такое

В PHP **всё — переменная**, а каждая переменная внутри движка представлена структурой `zval`.  
Это **контейнер**, который хранит тип, значение и счётчик ссылок.

В PHP 7+ она выглядит примерно так (в C):

```c
typedef struct _zval_struct {
    zend_value        value;     // само значение
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar type,      // тип (IS_LONG, IS_STRING и т.п.)
                zend_uchar type_flags,
                zend_uchar const_flags,
                zend_uchar reserved)
        } v;
    } u1;
    union {
        uint32_t next;           // для GC
        uint32_t cache_slot;
        uint32_t lineno;
    } u2;
} zval;
```

---

## Что хранит `zval`

|Поле|Назначение|
|---|---|
|`value`|Само значение (int, string, object и т.д.)|
|`type`|Константа типа (`IS_LONG`, `IS_STRING`, `IS_OBJECT`, ...)|
|`refcount` (внутри GC)|Сколько переменных указывает на этот zval|
|`is_ref`|Флаг «по ссылке»|
|`type_flags`|Всякие служебные отметки (например, что zval постоянный)|

---

## Пример: копирование переменной

```php
$a = 42;
$b = $a;
$b = 100;
```

После `$b = $a` обе переменные указывают **на один zval**, refcount = 2.  
После `$b = 100` у `$b` создаётся **новый zval**, а `$a` остаётся прежним.  
Это называется **copy-on-write (COW)** — экономия памяти до момента изменения.

---

## Пример: ссылки

```php
$a = 42;
$b =& $a;
$b = 100;
```

Теперь `$a` и `$b` делят **один и тот же zval**, refcount = 2, флаг `is_ref = 1`.  
Изменения видны в обеих переменных.

---

## Где это видно

Можно вызвать `debug_zval_dump()`:

```php
$a = "hi";
$b = $a;
debug_zval_dump($a, $b);
```

Вывод покажет refcount и is_ref.

---

## Почему это важно

1. **Производительность:** PHP экономит память за счёт copy-on-write.
    
2. **Расширения:** при написании C-модулей нужно следить за refcount и типами.
    
3. **Понимание поведения:** помогает объяснить «почему массив скопировался», «почему ссылка работает странно» и т.д.
    
4. **GC (сборщик мусора):** управляет zval через refcount и корневые буферы.
    

---

## Лирика

`zval` — это как маленький ящик с типом и числом друзей внутри.  
Пока друзья не предали его копированием, он экономит память.  
Потом кто-то делает `$b = 123`, и всё, дружба окончена, у каждого свой ящик.

---

## Полезные ссылки

- [PHP Internals Book – zvals](https://www.phpinternalsbook.com/php7/zvals/basic_structure.html)
    
- [zend_types.h в исходниках PHP](https://github.com/php/php-src/blob/master/Zend/zend_types.h)
    
- `debug_zval_dump()` в [PHP manual](https://www.php.net/manual/en/function.debug-zval-dump.php)
	
- [[Garbage Collector]]]

see also `lval`
