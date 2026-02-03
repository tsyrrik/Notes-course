## Вопрос: Жизненный цикл HTTP-запроса в Symfony
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
Request приходит в HttpKernel, проходит через событие (kernel.request), роутер выбирает контроллер, выполняется контроллер, формируется Response, проходят завершающие события (kernel.response/terminate), ответ отправляется.

## Ответ
Этапы:
1) client → front controller (`public/index.php`) создаёт Request.  
2) Kernel dispatches `kernel.request` (listeners могут изменить/создать Response).  
3) RouterListener мапит маршрут, аргументы резолвятся.  
4) Controller вызывается, возвращает Response (или данные, превращаемые в Response).  
5) `kernel.response` — обработка ответа (кэш, заголовки).  
6) Отправка ответа, `kernel.terminate` для пост-работы.

## Примеры

1. `kernel.request` редиректит неавторизованных.
2. `kernel.response` добавляет заголовок `X-Request-Id`.
3. `kernel.terminate` отправляет метрики после ответа.

## Доп. теория

1. Слушатели могут вернуть Response раньше контроллера.
2. События идут по порядку и влияют на финальный ответ.
