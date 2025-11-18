# Жизненный цикл HTTP-запроса в Symfony

Простыми словами: Request приходит в HttpKernel, проходит через событие (kernel.request), роутер выбирает контроллер, выполняется контроллер, формируется Response, проходят завершающие события (kernel.response/terminate), ответ отправляется.

Этапы:
1) client → front controller (`public/index.php`) создаёт Request.  
2) Kernel dispatches `kernel.request` (listeners могут изменить/создать Response).  
3) RouterListener мапит маршрут, аргументы резолвятся.  
4) Controller вызывается, возвращает Response (или данные, превращаемые в Response).  
5) `kernel.response` — обработка ответа (кэш, заголовки).  
6) Отправка ответа, `kernel.terminate` для пост-работы.
