# SOLID, KISS и YAGNI в 1C-Битрикс: практическое руководство для backend-разработчика
### Как писать поддерживаемый код в Bitrix без переусложнения и «магии»

> Формат: короткая теория + рабочие примеры на PHP + разбор типичных ошибок  
> Цель: показать, как применять SOLID, KISS и YAGNI в реальных Bitrix-проектах
> Для кого: junior/middle/senior PHP и 1C-Битрикс разработчики

---

## Главная идея за 30 секунд

Представь кухню в ресторане:

- **SOLID** = правильные роли (повар готовит, кассир принимает оплату, курьер доставляет).  
- **KISS** = не строим робот-кухню, если нужно просто пожарить яйцо.  
- **YAGNI** = не покупаем 5 печей “на будущее”, если заказов на одну.  

В Битриксе это особенно важно, потому что проекты часто растут быстро, и “временный код” живёт годами.

---

## Содержание

1. [SRP — Принцип единственной ответственности](#1-srp--принцип-единственной-ответственности-single-responsibility-principle)
2. [OCP — Принцип открытости/закрытости](#2-ocp--принцип-открытостизакрытости-openclosed-principle)
3. [LSP — Принцип подстановки Лисков](#3-lsp--принцип-подстановки-барбары-лисков-liskov-substitution-principle)
4. [ISP — Принцип разделения интерфейсов](#4-isp--принцип-разделения-интерфейса-interface-segregation-principle)
5. [DIP — Принцип инверсии зависимостей](#5-dip--принцип-инверсии-зависимостей-dependency-inversion-principle)
6. [KISS — Принцип простоты](#6-kiss--принцип-простоты-keep-it-simple-stupid)
7. [YAGNI — Принцип «вам это не понадобится»](#7-yagni--принцип-вам-это-не-понадобится-you-arent-gonna-need-it)

---

## 1) SRP — Принцип единственной ответственности (Single Responsibility Principle)

***Теория:***  
Принцип единственной ответственности говорит: у класса должна быть только одна зона ответственности.  
Если класс одновременно создает заказ, отправляет письма и пишет логи, он перегружен.  
Тогда любое изменение в одной части может случайно сломать другую.  
В Битрикс-проектах это частая причина “хрупкого” кода.  
Разделение обязанностей делает код понятнее, безопаснее и проще для поддержки.

**Кухня:** один повар отвечает за одну станцию, а не за всю кухню.


### Плохой пример

```php
<?php

final class OrderManager
{
    public function createOrder(array $orderData): int
    {
        if (empty($orderData['USER_ID'])) {
            throw new \InvalidArgumentException('USER_ID is required');
        }

        $orderId = $this->saveOrder($orderData);

        \CEvent::Send('ORDER_CREATED', 's1', [
            'ORDER_ID' => $orderId,
            'EMAIL' => $orderData['EMAIL'] ?? '',
        ]);

        file_put_contents(
            $_SERVER['DOCUMENT_ROOT'] . '/upload/order.log',
            date('c') . ' Order created: ' . $orderId . PHP_EOL,
            FILE_APPEND
        );

        return $orderId;
    }

    private function saveOrder(array $orderData): int
    {
        return 101;
    }
}
```




### Хороший пример

```php
<?php

final class OrderService
{
    public function create(array $orderData): int
    {
        if (empty($orderData['USER_ID'])) {
            throw new \InvalidArgumentException('USER_ID is required');
        }

        return 101;
    }
}

final class OrderNotifier
{
    public function sendCreated(int $orderId, string $email): void
    {
        \CEvent::Send('ORDER_CREATED', 's1', [
            'ORDER_ID' => $orderId,
            'EMAIL' => $email,
        ]);
    }
}

final class OrderLogger
{
    public function logCreated(int $orderId): void
    {
        file_put_contents(
            $_SERVER['DOCUMENT_ROOT'] . '/upload/order.log',
            date('c') . ' Order created: ' . $orderId . PHP_EOL,
            FILE_APPEND
        );
    }
}
```



---

## 2) OCP — Принцип открытости/закрытости (Open/Closed Principle)

***Теория:***  
Принцип открытости/закрытости означает: код должен быть открыт для расширения, но закрыт для изменения.  
Новую логику лучше добавлять новыми классами, а не переписывать старые рабочие блоки.  
Так снижается риск поломок в уже проверенном функционале.  
В Битриксе это особенно полезно для скидок, доставки, статусов и обработчиков.  
Итог: система растет без постоянных “хирургических” правок старого кода.

**Кухня:** новое блюдо добавили новым рецептом, старые рецепты не трогаем.


### Плохой пример

```php
<?php

final class DiscountService
{
    public function getPercent(string $userType): int
    {
        if ($userType === 'new') {
            return 0;
        }

        if ($userType === 'regular') {
            return 5;
        }

        if ($userType === 'vip') {
            return 15;
        }

        if ($userType === 'partner') {
            return 10;
        }

        return 0;
    }
}
```




### Хороший пример

```php
<?php

interface DiscountRuleInterface
{
    public function supports(string $userType): bool;

    public function getPercent(): int;
}

final class NewUserDiscountRule implements DiscountRuleInterface
{
    public function supports(string $userType): bool
    {
        return $userType === 'new';
    }

    public function getPercent(): int
    {
        return 0;
    }
}

final class RegularUserDiscountRule implements DiscountRuleInterface
{
    public function supports(string $userType): bool
    {
        return $userType === 'regular';
    }

    public function getPercent(): int
    {
        return 5;
    }
}

final class VipUserDiscountRule implements DiscountRuleInterface
{
    public function supports(string $userType): bool
    {
        return $userType === 'vip';
    }

    public function getPercent(): int
    {
        return 15;
    }
}

final class PartnerUserDiscountRule implements DiscountRuleInterface
{
    public function supports(string $userType): bool
    {
        return $userType === 'partner';
    }

    public function getPercent(): int
    {
        return 10;
    }
}

final class DiscountCalculator
{
    /**
     * @param DiscountRuleInterface[] $rules
     */
    public function __construct(private array $rules)
    {
    }

    public function getPercent(string $userType): int
    {
        foreach ($this->rules as $rule) {
            if ($rule->supports($userType)) {
                return $rule->getPercent();
            }
        }

        return 0;
    }
}
```



---

## 3) LSP — Принцип подстановки Барбары Лисков (Liskov Substitution Principle)

***Теория:***   
Принцип подстановки Лисков: объект дочернего класса должен без проблем заменять базовый тип.  
Если интерфейс обещает вернуть список, реализация не должна “внезапно” вести себя иначе.  
Вызывающий код должен получать предсказуемое поведение независимо от конкретной реализации.  
Нарушение этого принципа дает трудноуловимые баги в рантайме.  
Соблюдение LSP делает архитектуру надежной при замене модулей и источников данных.

**Кухня:** любой повар на позиции “суп” должен стабильно готовить суп по стандарту.


### Плохой пример

```php
<?php

interface ProductProviderInterface
{
    public function getList(): array;
}

final class BrokenProductProvider implements ProductProviderInterface
{
    public function getList(): array
    {
        throw new \RuntimeException('Always fails');
    }
}
```




### Хороший пример

```php
<?php

interface ProductProviderInterface
{
    public function getList(): array;
}

final class IblockProductProvider implements ProductProviderInterface
{
    public function getList(): array
    {
        $result = [];

        $res = \CIBlockElement::GetList(
            ['SORT' => 'ASC'],
            ['IBLOCK_ID' => 7, 'ACTIVE' => 'Y'],
            false,
            ['nTopCount' => 20],
            ['ID', 'NAME']
        );

        while ($row = $res->Fetch()) {
            $result[] = $row;
        }

        return $result;
    }
}
```



---

## 4) ISP — Принцип разделения интерфейса (Interface Segregation Principle)

***Теория:***  
Принцип разделения интерфейса говорит: не заставляй класс реализовывать лишние методы.  
Лучше сделать несколько небольших интерфейсов, чем один “комбайн”.  
Это уменьшает связанность и убирает пустые/фиктивные реализации.  
В Битриксе такой подход упрощает сервисы инфоблоков, заказов и интеграций.  
Код становится чище и легче в рефакторинге.

**Кухня:** кондитер не обязан заниматься доставкой и бухгалтерией.


### Плохой пример

```php
<?php

interface ContentManagerInterface
{
    public function add(array $fields): int;

    public function update(int $id, array $fields): bool;

    public function delete(int $id): bool;

    public function exportToXml(): string;

    public function sendNotification(array $payload): void;
}
```




### Хороший пример

```php
<?php

interface ContentCreatorInterface
{
    public function add(array $fields): int;
}

interface ContentUpdaterInterface
{
    public function update(int $id, array $fields): bool;
}

interface ContentRemoverInterface
{
    public function delete(int $id): bool;
}
```



---

## 5) DIP — Принцип инверсии зависимостей (Dependency Inversion Principle)

***Теория:***   
Принцип инверсии зависимостей: бизнес-логика должна зависеть от абстракций, а не от конкретных классов.  
Сервису важно “отправить уведомление”, а не “вызвать именно CEvent”.  
Это позволяет легко менять инфраструктуру без переписывания ядра логики.  
Тестировать такой код проще, потому что можно подставлять mock/fake-реализации.  
В итоге проект становится гибче и дешевле в сопровождении.

**Кухня:** важно, что заказ доставлен, а не какой конкретно курьер его вез.


### Плохой пример

```php
<?php

final class RegistrationService
{
    public function register(array $fields): int
    {
        $user = new \CUser();
        $userId = (int)$user->Add($fields);

        if ($userId <= 0) {
            throw new \RuntimeException($user->LAST_ERROR ?: 'Register error');
        }

        \CEvent::Send('NEW_USER', 's1', [
            'USER_ID' => $userId,
            'EMAIL' => $fields['EMAIL'] ?? '',
        ]);

        return $userId;
    }
}
```




### Хороший пример

```php
<?php

interface UserGatewayInterface
{
    public function create(array $fields): int;
}

interface NotifierInterface
{
    public function send(string $event, array $payload): void;
}

final class BitrixUserGateway implements UserGatewayInterface
{
    public function create(array $fields): int
    {
        $user = new \CUser();
        $userId = (int)$user->Add($fields);

        if ($userId <= 0) {
            throw new \RuntimeException($user->LAST_ERROR ?: 'Register error');
        }

        return $userId;
    }
}

final class BitrixNotifier implements NotifierInterface
{
    public function send(string $event, array $payload): void
    {
        \CEvent::Send($event, 's1', $payload);
    }
}

final class RegistrationService
{
    public function __construct(
        private UserGatewayInterface $userGateway,
        private NotifierInterface $notifier
    ) {
    }

    public function register(array $fields): int
    {
        $userId = $this->userGateway->create($fields);

        $this->notifier->send('NEW_USER', [
            'USER_ID' => $userId,
            'EMAIL' => $fields['EMAIL'] ?? '',
        ]);

        return $userId;
    }
}
```



---

## 6) KISS — Принцип простоты (Keep It Simple, Stupid)

***Теория:***  
Принцип простоты: выбирай самое простое решение, которое качественно решает задачу.  
Не нужно строить сложную архитектуру там, где хватает стандартного механизма Битрикса.  
Сложность должна появляться только при реальной необходимости.  
Простые решения быстрее запускать, проще объяснять и легче поддерживать.  
Это снижает риск ошибок и ускоряет работу команды.

**Кухня:** не строим робот-кухню, если нужно просто пожарить яйцо.


### Плохой пример

```php
<?php

final class ProductQueryBuilder
{
    private array $select = [];
    private array $filter = [];
    private array $sort = [];
    private int $limit = 20;

    public function select(array $fields): self
    {
        $this->select = $fields;
        return $this;
    }

    public function filter(array $filter): self
    {
        $this->filter = $filter;
        return $this;
    }

    public function sort(array $sort): self
    {
        $this->sort = $sort;
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function build(): array
    {
        return [
            'select' => $this->select,
            'filter' => $this->filter,
            'sort' => $this->sort,
            'limit' => $this->limit,
        ];
    }
}
```




### Хороший пример

```php
<?php

$res = \CIBlockElement::GetList(
    ['SORT' => 'ASC'],
    ['IBLOCK_ID' => 7, 'ACTIVE' => 'Y'],
    false,
    ['nTopCount' => 20],
    ['ID', 'NAME', 'DETAIL_PAGE_URL']
);

$products = [];

while ($row = $res->Fetch()) {
    $products[] = $row;
}
```



---

## 7) YAGNI — Принцип «Вам это не понадобится» (You Aren’t Gonna Need It)

***Теория:***  
Принцип YAGNI: не реализуй функциональность, которая сейчас не нужна бизнесу.  
Код “на будущее” часто никогда не используется, но усложняет систему уже сегодня.  
Лишние абстракции увеличивают стоимость разработки и шанс багов.  
Лучше сделать ровно под текущую задачу и расширить позже по факту потребности.  
Так команда экономит время и сохраняет фокус на результате.

**Кухня:** не покупаем 5 печей “на будущее”, если заказов на одну.


### Плохой пример

```php
<?php

interface CrmSenderInterface
{
    public function sendLead(array $lead): void;
}

final class AmoCrmSender implements CrmSenderInterface
{
    public function sendLead(array $lead): void
    {
        // send to amoCRM
    }
}

final class Bitrix24CrmSender implements CrmSenderInterface
{
    public function sendLead(array $lead): void
    {
        // send to Bitrix24 CRM
    }
}

final class SalesforceCrmSender implements CrmSenderInterface
{
    public function sendLead(array $lead): void
    {
        // send to Salesforce
    }
}
```




### Хороший пример

```php
<?php

final class AmoCrmSender
{
    public function sendLead(array $lead): void
    {
        // Реальная текущая задача: отправка только в amoCRM
    }
}
```



---

## Финальный чек на каждый новый класс

- У класса одна ответственность?
- Новое поведение добавляю, а не переписываю старую логику?
- Реализация интерфейса предсказуема при подстановке?
- Интерфейс не перегружен лишними методами?
- Я завишу от абстракций, а не от конкретных деталей?
- Решение достаточно простое?
- Не пишу лишнее “на будущее”?

---

**Авторский принцип:** польза, экономия времени, усиление решений. Бесполезное недопустимо.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
