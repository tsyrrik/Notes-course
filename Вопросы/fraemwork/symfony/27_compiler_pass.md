## Вопрос: Compiler Pass
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
