
**Event-driven design** — подход, где взаимодействие между компонентами строится через **события**, а не прямые вызовы.  
Каждый компонент **реагирует на события**, а не **дергает других напрямую**.

---

### 1. Что такое событие

**Событие (event)** — это факт, который уже произошёл.  
Оно описывает _что случилось_, а не _что нужно сделать_.

Пример:

```php
// Плохо — команда (command)
$user->sendWelcomeEmail();

// Хорошо — событие
$eventBus->dispatch(new UserRegisteredEvent($userId));
```

`UserRegisteredEvent` сообщает, что пользователь зарегистрировался.  
А кто и как отреагирует — решается вне исходного кода модуля регистрации.

---

### 2. Основная идея

Компоненты системы **не знают друг о друге**.  
Они просто публикуют события — и другие слушатели решают, нужно ли им реагировать.

**Publisher (отправитель)** — сообщает факт.  
**Subscriber (подписчик)** — реагирует на него.

Так снижается связанность, появляется гибкость, расширяемость и возможность добавлять новые реакции без трогания исходного кода.

---

### 3. Пример

```php
class RegisterUserHandler
{
    public function __construct(private EventBus $bus) {}

    public function handle(string $email, string $password): void
    {
        $user = new User($email, $password);
        $user->save();

        $this->bus->dispatch(new UserRegisteredEvent($user->id));
    }
}
```

```php
class SendWelcomeEmailListener
{
    public function __invoke(UserRegisteredEvent $event): void
    {
        $user = User::find($event->userId);
        Mail::to($user->email)->send(new WelcomeMail());
    }
}
```

Теперь можно спокойно добавить ещё одного слушателя,  
например `CreateUserAnalyticsRecordListener`, не трогая регистрацию.

---

### 4. Event Bus и брокеры

Внутри приложения событие может быть **в памяти** (через шину Symfony, Laravel EventDispatcher и т. д.).  
В распределённых системах — через **брокеров сообщений**:  
RabbitMQ, Kafka, Redis Streams, SQS и т. п.

Они делают события **асинхронными**, что особенно важно для масштабируемости.

---

### 5. Типы событий

|Тип|Пример|Описание|
|---|---|---|
|**Domain Event**|`UserRegistered`|произошёл факт в предметной области|
|**Integration Event**|`OrderPaid` (в шину Kafka)|событие для внешних сервисов|
|**System Event**|`CacheInvalidated`|внутреннее тех. событие|

---

### 6. Связь с другими паттернами

- **Command** — просит _сделать что-то_ (в будущем).
    
- **Event** — сообщает, _что уже произошло_.
    
- **CQRS** — часто сочетается с EDD: Commands изменяют состояние, Events рассылаются после изменений.
    
- **Event Sourcing** — хранит все события как источник истины.
    

