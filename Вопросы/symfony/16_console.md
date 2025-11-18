# Консоль и команды

Простыми словами: Console компонент позволяет создавать свои CLI-команды.

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
