
- Коммить `composer.lock`, но не `vendor/`
    
- Не используй `*` и `dev-master` в `require`
    
- Для библиотек добавляй:
    
    ```json
    "minimum-stability": "dev",
    "prefer-stable": true
    ```
    
- В CI-пайплайне:
    
    ```bash
    composer validate
    composer check-platform-reqs
    composer audit
    composer outdated --direct --minor-only
    ```
    
- В проде:
    
    ```bash
    composer install --no-dev --optimize-autoloader --no-progress
    ```
    