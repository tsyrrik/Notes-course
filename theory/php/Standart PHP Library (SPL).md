[Официальная документация](https://php.net/manual/ru/book.spl.php)

Включает в себя набор базовых реализаций структур данных, итераторов, интерфейсов, исключений и функций. Появилась в версии PHP 5.0.0, после версии PHP 5.3.0 поставляется вместе с ядром PHP.

## SPL: Классы и структуры данных

* [ArrayObject](https://www.php.net/manual/ru/class.arrayobject.php) - позволяет объектам работать как массивы. ООП обёртка для массивов
* [SplFixedArray](https://www.php.net/manual/ru/class.splfixedarray.php) - имеет фиксированную длину, а в качестве индексов могут выступать только целочисленные значения. Преимущество заключается в более быстрой обработке массива, за счёт того, что он не реализует hash-map.
> по факту массивы в PHP реализованы по принципу hash-map. т.е. при создании ключа в массиве, этот ключ проходит через специальную hash-функцию, которая понимает в какой определённой ячейки памяти находится значение.
* [SplMaxHeap/SplMinHeap](https://www.php.net/manual/ru/class.splmaxheap.php) - предоставляет основные функциональные возможности кучи. 
* [SplFileObject](https://www.php.net/manual/ru/class.splfileobject.php) - предоставляет объектно-ориентированный интерфейс для файла
* [SplFileInfo](https://www.php.net/manual/ru/class.splfileinfo.php) - предлагает высокоуровневый объектно-ориентированный интерфейс к информации для отдельного файла

## SPL: Итераторы

* [ArrayIterator](https://www.php.net/manual/ru/class.arrayiterator.php) - итератор для простого массива. Позволяет сбрасывать и модифицировать значения и ключи в процессе итерации по массивам и объектам
* [DirectoryIterator](https://www.php.net/manual/ru/class.directoryiterator.php) - предоставляет простой интерфейс для просмотра содержимого каталогов файловой системы
* [EmptyIterator](https://www.php.net/manual/en/class.emptyiterator.php) - пустой итератор, реализация шаблона Null object
* [CallbackFilterIterator](https://www.php.net/manual/ru/class.callbackfilteriterator.php) - позволяет отфильтравать элементы другого итератора

## SPL: Исключения

* [DomainException](https://www.php.net/manual/ru/class.domainexception.php) - исключение, если значение не соответствует определенным допустимым данным домена
* [LogicException](https://www.php.net/manual/ru/class.logicexception.php) - исключение, которое представляет ошибку в логике программы. Такой тип исключений должен непосредственно привести к исправлениям в вашем коде. 
* [RuntimeException](https://www.php.net/manual/ru/class.runtimeexception.php) - исключение в случае ошибки, которая может быть найдена только во время выполнения
* [InvalidArgumentException](https://www.php.net/manual/ru/class.invalidargumentexception.php) - исключение, если аргумент не соответствует ожидаемому типу
* [BadMethodCallException](https://www.php.net/manual/ru/class.badmethodcallexception.php) - исключение, если callback-функция относится к неопределенному методу или если некоторые аргументы отсутствуют

## SPL: Функции

* [spl_autoload_register](https://www.php.net/manual/ru/function.spl-autoload-register.php) - Регистрирует заданную функцию в качестве реализации метода __autoload() 
* [spl_object_id](https://www.php.net/manual/ru/function.spl-object-id.php) - возвращает уникальный ID объекта
* [spl_object_hash](https://www.php.net/manual/ru/function.spl-object-hash.php) - возвращает уникальный hash объекта
* [iterator_to_array](https://www.php.net/manual/ru/function.iterator-to-array.php) - копирует итератор в массив