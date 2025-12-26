## Вопрос: Консоль и команды
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
