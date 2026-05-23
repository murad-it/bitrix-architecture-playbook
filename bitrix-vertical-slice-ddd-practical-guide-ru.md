# Vertical Slice и DDD в 1C-Битрикс: практическая архитектура для реальных проектов
### Как разделять Domain, Application, Infrastructure и Presentation без «слоёного хаоса»

> Формат: суть + примеры на PHP/Bitrix + формулировки для собеседований
> Цель: применять DDD и Vertical Slice в Bitrix-коде так, чтобы изменения были локальными и безопасными
> Для кого: backend-разработчики и техлиды на 1C-Битрикс

---

## Главная идея простыми словами

Представь кухню в ресторане:

- **Vertical Slice (вертикальный срез)** = делаем не “отдел бэкенда отдельно, отдел БД отдельно”, а собираем одну фичу целиком от входа до результата.
- **DDD-слои (Domain-Driven Design)** = у каждого уровня своя роль:  
  `Domain (Предметная область)` — правила бизнеса,  
  `Application (Сценарий)` — последовательность действий,  
  `Infrastructure (Инфраструктура)` — техническая реализация,  
  `Presentation (Представление)` — как пользователь/API взаимодействует с системой.

В Битриксе это важно особенно сильно: проекты растут быстро, и без структуры логика склеивается в один “ком”.

---

## Что такое Vertical Slice (вертикальный срез)

**Короткая суть:**  
Vertical Slice — это когда мы организуем код вокруг **конкретной бизнес-фичи**, а не вокруг технических слоев-папок.

**Как обычно бывает плохо:**  
`Controllers`, `Services`, `Repositories` лежат отдельно, и одна фича размазана по 10 папкам.

**Как лучше:**  
Фича “Создать заказ” лежит в одном месте, внутри нее уже есть нужные слои.

### Плохой вариант структуры
```text
local/src/
  Controllers/
  Services/
  Repositories/
  Helpers/
```

### Хороший вариант структуры
```text
local/modules/acme.shop/src/
  Order/
    Create/
      Presentation/
      Application/
      Domain/
      Infrastructure/
```

**Почему это удобно:**
1. Изменения локальные.
2. Быстрее разбираться в коде.
3. Ниже риск сломать соседние фичи.

---

## Что такое DDD (предметно-ориентированное проектирование)

**Короткая суть:**  
DDD (Domain-Driven Design) — подход, где в центре не таблицы и не фреймворк, а **бизнес-смысл**.  
Сначала мы фиксируем правила предметной области, потом думаем о технической реализации.

**Одна сильная фраза для интервью:**  
“DDD помогает отделить бизнес-правила от технических деталей, чтобы система менялась безопасно и предсказуемо”.

---

## DDD-слои: смысл каждого слоя

## 1) Domain (Предметная область)

**Что это:**  
Ядро бизнес-правил. Здесь определяется, что “правильно” и “неправильно” для предметной области.

**Что хранить в Domain:**
- сущности (Entity),
- value objects,
- бизнес-ограничения,
- инварианты.

**Что нельзя тащить в Domain:**
- `CIBlockElement`, `CUser`, SQL, HTTP, шаблоны.

**Плохой пример (Domain знает про Bitrix API):**
```php
<?php

final class Order
{
    public function save(array $data): int
    {
        $el = new \CIBlockElement();
        return (int)$el->Add(['IBLOCK_ID' => 12, 'NAME' => 'Order']);
    }
}
```

**Хороший пример (чистые бизнес-правила):**
```php
<?php

final class Order
{
    public function __construct(
        private int $userId,
        private float $total
    ) {
        if ($this->userId <= 0) {
            throw new \InvalidArgumentException('User ID must be positive');
        }

        if ($this->total <= 0) {
            throw new \InvalidArgumentException('Total must be positive');
        }
    }

    public function userId(): int
    {
        return $this->userId;
    }

    public function total(): float
    {
        return $this->total;
    }
}
```

---

## 2) Application (Слой сценариев / use case)

**Что это:**  
Слой, который управляет шагами конкретного сценария: создать заказ, отменить заказ, зарегистрировать пользователя.

**Роль:**  
Он не придумывает бизнес-смысл, а **оркестрирует**: принять команду -> вызвать Domain -> сохранить -> вернуть результат.

**Плохой пример (в одном месте все сразу):**
```php
<?php

final class CreateOrderController
{
    public function execute(array $request): array
    {
        if ((float)($request['TOTAL'] ?? 0) <= 0) {
            return ['status' => 'error'];
        }

        $el = new \CIBlockElement();
        $id = $el->Add(['IBLOCK_ID' => 12, 'NAME' => 'Order']);

        return ['status' => 'ok', 'id' => (int)$id];
    }
}
```

**Хороший пример (сценарий отдельно):**
```php
<?php

interface OrderRepositoryInterface
{
    public function save(Order $order): int;
}

final class CreateOrderCommand
{
    public function __construct(
        public readonly int $userId,
        public readonly float $total
    ) {
    }
}

final class CreateOrderHandler
{
    public function __construct(private OrderRepositoryInterface $orders)
    {
    }

    public function handle(CreateOrderCommand $command): int
    {
        $order = new Order($command->userId, $command->total);

        return $this->orders->save($order);
    }
}
```

---

## 3) Infrastructure (Инфраструктурный слой)

**Что это:**  
Технические детали: Bitrix API, базы данных, внешние HTTP-сервисы, очереди, файловая система.

**Роль:**  
Реализовать интерфейсы из Application/Domain и не тянуть бизнес-решения на себя.

**Плохой пример (инфраструктура решает бизнес-политику):**
```php
<?php

final class BitrixOrderRepository
{
    public function save(array $data): int
    {
        if (($data['TOTAL'] ?? 0) > 100000) {
            throw new \RuntimeException('Limit exceeded');
        }

        $el = new \CIBlockElement();
        return (int)$el->Add(['IBLOCK_ID' => 12, 'NAME' => 'Order']);
    }
}
```

**Хороший пример (только техническая реализация):**
```php
<?php

final class BitrixOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): int
    {
        $el = new \CIBlockElement();

        $id = $el->Add([
            'IBLOCK_ID' => 12,
            'NAME' => 'Order for user ' . $order->userId(),
            'PROPERTY_VALUES' => [
                'USER_ID' => $order->userId(),
                'TOTAL' => $order->total(),
            ],
        ]);

        if (!$id) {
            throw new \RuntimeException('Failed to save order');
        }

        return (int)$id;
    }
}
```

---

## 4) Presentation (Слой представления)

**Что это:**  
Точка входа/выхода: контроллер, Bitrix-компонент, ajax endpoint, API-ответ.

**Роль:**  
Принять запрос, передать в Application, вернуть ответ.  
Без тяжелой бизнес-логики и без прямого “ручного” хранения сущностей.

**Плохой пример:**
```php
<?php

final class CreateOrderController
{
    public function execute(array $request): array
    {
        if ((float)($request['TOTAL'] ?? 0) > 100000) {
            return ['status' => 'error', 'message' => 'Limit exceeded'];
        }

        $el = new \CIBlockElement();
        $id = $el->Add(['IBLOCK_ID' => 12, 'NAME' => 'Order']);

        return ['status' => 'ok', 'id' => (int)$id];
    }
}
```

**Хороший пример:**
```php
<?php

final class CreateOrderController
{
    public function __construct(private CreateOrderHandler $handler)
    {
    }

    public function execute(array $request): array
    {
        try {
            $command = new CreateOrderCommand(
                userId: (int)($request['USER_ID'] ?? 0),
                total: (float)($request['TOTAL'] ?? 0)
            );

            $orderId = $this->handler->handle($command);

            return [
                'status' => 'ok',
                'order_id' => $orderId,
            ];
        } catch (\Throwable $e) {
            return [
                'status' => 'error',
                'message' => $e->getMessage(),
            ];
        }
    }
}
```

---

## Как объяснить это на интервью (готовые формулировки)

### Короткая версия (20 секунд)
“Я использую DDD, чтобы разделять бизнес-правила и технические детали: Domain отвечает за смысл, Application за сценарий, Infrastructure за интеграции и хранение, Presentation за вход/выход. В связке с Vertical Slice это дает локальные и безопасные изменения.”

### Средняя версия (40-60 секунд)
“Для меня DDD — это способ держать бизнес-логику чистой и независимой от фреймворка. В Domain лежат правила предметной области, в Application — use case-оркестрация, Infrastructure реализует доступ к Bitrix и внешним системам, Presentation работает как тонкий слой доставки. Я группирую это в vertical slice по фичам, чтобы команда могла быстро менять одну функцию без каскадных поломок.”

### Сильная версия (если спросят “зачем бизнесу?”)
“Это снижает стоимость изменений: бизнес-правила меняются в одном месте, интеграции меняются отдельно, UI/API отдельно. Меньше регрессий, быстрее ревью, проще онбординг новых разработчиков.”

---

## Мини-шпаргалка: где что держать в Bitrix

- `Domain` — сущности, value objects, политики, инварианты.
- `Application` — команды, хендлеры, use cases, порты (интерфейсы).
- `Infrastructure` — `CIBlockElement`, `CUser`, ORM, API-клиенты, реализация портов.
- `Presentation` — `component.php`, контроллеры, ajax handlers, формат ответа.

---

## Частые ошибки (и как поправить)

1. **Ошибка:** бизнес-логика в `component.php`.  
   **Исправление:** вынести сценарий в `Application Handler`.

2. **Ошибка:** Domain дергает Bitrix API.  
   **Исправление:** вынести это в `Infrastructure Repository`.

3. **Ошибка:** один “мега-сервис” на всю систему.  
   **Исправление:** резать на vertical slices по фичам.

4. **Ошибка:** интерфейсы есть, но ими не пользуются.  
   **Исправление:** Application зависит от интерфейса, Infrastructure его реализует.

---

## Финальный смысл в одной строке

**DDD + Vertical Slice = бизнес-смысл отделен от технической рутины, а каждая фича развивается локально, быстро и безопасно.**

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
