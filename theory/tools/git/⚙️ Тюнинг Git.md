

Git гибче, чем думают.  
Например, можно:

- выбрать алгоритм диффов (см. [Habr: Настройка Git под себя](https://habr.com/ru/articles/886538/));
    
- уйти от `git push --set-upstream origin ...`
    
- сделать **автоматический push** после каждого коммита (`git config --global push.autoSetupRemote true`);
    
- включить **естественную сортировку тегов** (`git tag --sort=-version:refname`);
    
- задать **pre-commit hooks** и alias-команды.
    
