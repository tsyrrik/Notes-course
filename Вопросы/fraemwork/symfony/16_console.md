## Вопрос: Консоль и команды
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Console компонент позволяет создавать свои CLI-команды.

## Ответ
```bash
php bin/console list
php bin/console make:command app:hello
```
```php
class HelloCommand extends Command {
    protected static $defaultName = 'app:hello';
    protected function execute(InputInterface $input, OutputInterface $output): int {
        $output->writeln('Hello');
        return Command::SUCCESS;
    }
}
```

## Примеры

1. `php bin/console make:command app:hello`.
2. `php bin/console cache:clear` в prod.
3. Команда для nightly‑отчёта через cron.

## Доп. теория

1. Команды — сервисы контейнера и поддерживают autowiring.
2. Для тяжёлых задач лучше запускать через Messenger/queue.
