## Вопрос: Compiler Pass
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
хук на этапе компиляции контейнера DI — позволяет модифицировать определения сервисов, находить теги и собирать их.

## Ответ
```php
class CustomCompilerPass implements CompilerPassInterface {
    public function process(ContainerBuilder $container): void {
        $tagged = $container->findTaggedServiceIds('app.handler');
        foreach ($tagged as $id => $tags) {
            // зарегистрировать в агрегаторе, вызвать addHandler и т.п.
        }
    }
}
```

Подключение: зарегистрировать в `Kernel::build()` или в bundle (для библиотек). Используйте, когда нужно изменить контейнер программно.

## Примеры

1. Сбор сервисов с тегом `app.handler` в реестр.
2. Добавление аргумента в определение сервиса.
3. Автосбор стратегий для `StrategyRegistry`.

## Доп. теория

1. Compiler Pass выполняется при сборке контейнера, а не во время запроса.
2. Для простых случаев хватает `TaggedIterator` без Compiler Pass.
