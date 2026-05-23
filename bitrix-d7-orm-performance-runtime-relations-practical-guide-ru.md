# D7 ORM в 1C-Битрикс без иллюзий: как писать быстрые запросы, а не красивый тормоз
### Практическое руководство по ORM, runtime relation, JOIN, индексам MySQL и ускорению production-проектов

> Формат: боль -> антипаттерн -> рабочий паттерн -> чем проверить  
> Цель: использовать D7 ORM как инструмент ускорения, а не как «модный слой»  
> Для кого: middle/senior PHP-разработчики, техлиды, архитекторы на 1C-Битрикс

---

## Почему эта статья важна

На Bitrix-проектах спор «ORM или старый API» обычно заканчивается не идеологией, а метриками.
Если запросы спроектированы плохо, тормозить будет и ORM, и legacy-код.
Если запросы спроектированы правильно, D7 ORM даёт предсказуемость, читабельность и нормальную скорость.

Ключевая мысль простая: D7 ORM не ускоряет проект сам по себе. Ускоряет инженерная дисциплина.

---

## Где чаще всего ломается производительность в D7

1. Выбираем лишние поля и свойства «на будущее».
2. Плодим N+1 внутри циклов.
3. Делаем runtime relation без понимания SQL, который реально генерируется.
4. Добавляем индексы «на глаз», без `EXPLAIN`.
5. Включаем кэш без стратегии инвалидации.

---

## 1) D7 ORM: полезная модель, если думать SQL-ом

В D7 сущность (`DataManager`) это не «магия», а обёртка над таблицей с понятной картой полей и связей.
Нужно держать в голове, что любой `getList()` в итоге превращается в SQL, который надо уметь читать и дебажить.

```php
<?php
use Bitrix\Iblock\ElementTable;

$rows = ElementTable::getList([
    'select' => ['ID', 'NAME', 'IBLOCK_ID'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'order'  => ['ID' => 'DESC'],
    'limit'  => 20,
])->fetchAll();
```

Хороший тон: писать запрос так, чтобы его понимал не только автор, но и следующий разработчик через 6 месяцев.

---

## 2) Антипаттерн №1: «Давайте сразу всё выберем»

### Плохо

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['*'],
    'filter' => ['=IBLOCK_ID' => 7],
])->fetchAll();
```

### Почему это больно

- растут память и время на сериализацию;
- увеличивается нагрузка на БД;
- в ревью непонятно, какие данные реально нужны.

### Хорошо

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['ID', 'NAME', 'CODE', 'PREVIEW_PICTURE'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'order'  => ['SORT' => 'ASC', 'ID' => 'DESC'],
    'limit'  => 30,
])->fetchAll();
```

---

## 3) Антипаттерн №2: N+1 на свойствах и ценах

Главная ошибка: в цикле по элементам делать дополнительные запросы к ценам/свойствам.

### Паттерн без боли

1. Получаем список ID элементов.
2. Пакетно дочитываем связанные данные (`PriceTable`, `ElementPropertyTable`).
3. Собираем map в PHP и мержим в read-model.

```php
<?php
use Bitrix\Catalog\PriceTable;
use Bitrix\Iblock\ElementPropertyTable;
use Bitrix\Iblock\ElementTable;

$elements = ElementTable::getList([
    'select' => ['ID', 'NAME'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'limit'  => 100,
])->fetchAll();

$ids = array_map(static fn(array $row): int => (int)$row['ID'], $elements);

$prices = PriceTable::getList([
    'select' => ['PRODUCT_ID', 'PRICE'],
    'filter' => ['@PRODUCT_ID' => $ids, '=CATALOG_GROUP_ID' => 1],
])->fetchAll();

$props = ElementPropertyTable::getList([
    'select' => ['IBLOCK_ELEMENT_ID', 'IBLOCK_PROPERTY_ID', 'VALUE'],
    'filter' => ['@IBLOCK_ELEMENT_ID' => $ids],
])->fetchAll();
```

---

## 4) Runtime relation: мощно, но только если контролируешь JOIN

Runtime-поля закрывают много боевых задач, когда связь не описана в `getMap()` или нужна локально в одном запросе.

```php
<?php
use Bitrix\Main\ORM\Fields\Relations\Reference;
use Bitrix\Main\ORM\Query\Join;
use Bitrix\Sale\Internals\OrderPropsValueTable;
use Bitrix\Sale\Internals\OrderTable;

$orders = OrderTable::getList([
    'select' => ['ID', 'DATE_INSERT', 'PROP_VALUE.VALUE'],
    'filter' => ['!STATUS_ID' => ['C', 'RS']],
    'runtime' => [
        new Reference(
            'PROP_VALUE',
            OrderPropsValueTable::class,
            Join::on('this.ID', 'ref.ORDER_ID')
        ),
    ],
    'limit' => 50,
])->fetchAll();
```

ВНИМАНИЕ: любой runtime relation проверяй через SQL-трекер. Часто запрос выглядит красиво в PHP и плохо в БД.

---

## 5) JOIN-стратегия: LEFT по умолчанию не всегда лучший выбор

Если тебе нужны только строки с гарантированным соответствием в связанной таблице, часто выгоднее `INNER JOIN`.
Для опциональных данных — `LEFT JOIN`.
Критерий выбора: не «как привычно», а какой план исполнения дешевле.

```php
<?php
use Bitrix\Main\ORM\Fields\Relations\Reference;
use Bitrix\Main\ORM\Query\Join;

new Reference(
    'FILE',
    \Bitrix\Main\FileTable::class,
    Join::on('this.FILE_ID', 'ref.ID')
)->configureJoinType(Join::TYPE_INNER);
```

---

## 6) Индексы MySQL: ускоряют ORM так же, как и чистый SQL

D7 ORM не отменяет базу данных. Если фильтруешь по полю без индекса, ORM не спасёт.

### Правильный порядок

1. Снять медленные запросы.
2. Прогнать `EXPLAIN`.
3. Добавить индекс под фактический `WHERE`/`JOIN`.
4. Снова проверить план и время.

```sql
CREATE INDEX ix_ibep_prop_valnum
ON b_iblock_element_property (IBLOCK_PROPERTY_ID, VALUE_NUM);
```

ВНИМАНИЕ: индексы в проде только после проверки на копии базы и согласования с DBA.

---

## 7) ORM-кэш: использовать можно, но без самообмана

ORM поддерживает кэш в `getList`, но это не серебряная пуля.

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['ID', 'NAME'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'cache'  => [
        'ttl' => 3600,
        'cache_joins' => true,
    ],
])->fetchAll();
```

Что важно:

- кэшировать только запросы со стабильной бизнес-семантикой;
- понимать, когда и чем данные инвалидируются;
- при отладке производительности временно отключать кэш, чтобы видеть реальную стоимость запроса.

---

## 8) Отладка и замеры: без них «оптимизация» бесполезна

Минимальный набор для D7/Bitrix:

1. `\Bitrix\Main\Diag\SqlTracker` — понять, что реально уходит в БД.
2. Bitrix Performance Monitor — видеть горячие места.
3. `EXPLAIN` на ключевых запросах.
4. Таблица «до/после» по метрикам.

```php
<?php
\Bitrix\Main\Diag\SqlTracker::getInstance()->start();
// код с ORM-запросами
$queries = \Bitrix\Main\Diag\SqlTracker::getInstance()->getQueries();
```

---

## Мини-кейс: как выглядит нормальный результат

| Метрика | До | После |
|---|---:|---:|
| SQL-запросы на страницу каталога | 246 | 58 |
| P95 server time, ms | 1840 | 520 |
| Peak memory, MB | 29 | 14 |

Почему сработало:

- убрали N+1;
- сузили `select`;
- перестроили runtime JOIN;
- добавили правильный индекс под фильтр.

---

## Финальный чек-лист для code review (D7 ORM)

1. В `select` только нужные поля?
2. Нет N+1 внутри циклов?
3. Runtime relation обоснован и проверен по SQL?
4. Есть индексы под ключевые фильтры/joins?
5. Кэш не ломает актуальность данных?
6. Есть измерения «до/после», а не только «ощущения»?

---

## Вывод

Если использовать D7 ORM как инженерный инструмент, он даёт чистую архитектуру и нормальную скорость.
Если использовать ORM как «магический слой без SQL-мышления», проект неизбежно упрётся в производительность.

Сильный подход для Bitrix один: читаемые запросы, осознанные JOIN, корректные индексы, проверяемые метрики.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
