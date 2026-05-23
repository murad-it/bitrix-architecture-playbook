# D7 ORM в 1C-Битрикс на production: как писать быстрый код, а не красивую деградацию
### Глубокое практическое руководство: Query API, runtime relation, JOIN-стратегии, индексы MySQL, кэширование и диагностика производительности

> Формат: контекст -> теория -> антипаттерн -> рабочий паттерн -> верификация метриками  
> Цель: дать инженерную систему работы с D7 ORM, которая реально ускоряет проект  
> Для кого: middle/senior разработчики, техлиды, архитекторы на 1C-Битрикс

---

## Зачем вообще эта статья

Если коротко: D7 ORM в Bitrix это не «про модно», а про управляемость изменений и снижение риска регрессий.  
Но только при одном условии: команда понимает, какой SQL рождается под капотом и как этот SQL исполняется в MySQL.

На практике я чаще вижу две крайности:

1. `Мы пишем на старом API, ORM медленный`.  
2. `Мы пишем на ORM, значит всё автоматически хорошо`.

Обе позиции ошибочные. Скорость и стабильность определяются не API, а качеством проектирования выборок, индексов, кэш-стратегии и контролем метрик.

---

## Базовая инженерная модель (прежде чем писать код)

Перед любым «ускорением» фиксируем baseline:

1. `P95 server time` целевого сценария (например, категория каталога).
2. Количество SQL-запросов и топ-10 по времени.
3. Пиковая память PHP.
4. Количество внешних HTTP-вызовов в рамках запроса.
5. Время в БД vs время в PHP.

Без baseline любая оптимизация превращается в субъективное ощущение.

---

## Как устроен D7 ORM в реальности

В D7 ключевые сущности для чтения данных:

1. `DataManager` (таблица/сущность).
2. `getList()` / `query()` (построение запроса).
3. `runtime` (временные поля/связи/выражения).
4. `Reference` и `ExpressionField` (JOIN и вычисления).

Важно: D7 ORM не изолирует тебя от SQL. Он меняет форму записи запроса, но ответственность за его стоимость остаётся на разработчике.

---

## Часть 1. Топ-ошибки в D7 ORM, которые делают проект медленным

## 1) Избыточный `select`

### Антипаттерн

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['*'],
    'filter' => ['=IBLOCK_ID' => 7],
])->fetchAll();
```

### Почему это плохо

- Лишняя нагрузка на БД и сеть БД->PHP.
- Рост памяти и времени гидрации результата.
- Ревью не может проверить осмысленность данных.

### Рабочий паттерн

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['ID', 'NAME', 'CODE', 'IBLOCK_ID', 'PREVIEW_PICTURE'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'order'  => ['SORT' => 'ASC', 'ID' => 'DESC'],
    'limit'  => 30,
])->fetchAll();
```

Правило: выбираем только поля, которые реально участвуют в бизнес-решении или рендере.

---

## 2) N+1 по ценам, свойствам, разделам

### Антипаттерн

Один базовый запрос по элементам + дополнительные запросы в цикле.

### Рабочий паттерн

1. Получить элементы.
2. Вытащить их ID.
3. Пакетно загрузить цены/свойства/связи.
4. Собрать read model.

```php
<?php
use Bitrix\Catalog\PriceTable;
use Bitrix\Iblock\ElementPropertyTable;
use Bitrix\Iblock\ElementTable;

$elements = ElementTable::getList([
    'select' => ['ID', 'NAME', 'IBLOCK_ID'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'limit'  => 100,
])->fetchAll();

$ids = array_map(static fn(array $row): int => (int)$row['ID'], $elements);

$priceRows = PriceTable::getList([
    'select' => ['PRODUCT_ID', 'PRICE'],
    'filter' => ['@PRODUCT_ID' => $ids, '=CATALOG_GROUP_ID' => 1],
])->fetchAll();

$propRows = ElementPropertyTable::getList([
    'select' => ['IBLOCK_ELEMENT_ID', 'IBLOCK_PROPERTY_ID', 'VALUE'],
    'filter' => ['@IBLOCK_ELEMENT_ID' => $ids],
])->fetchAll();
```

Результат: вместо сотен мелких запросов получаешь несколько крупных и предсказуемых.

---

## 3) Runtime relation «на глаз»

Runtime relation полезен, когда связь нужна локально, но он легко создаёт тяжёлые JOIN.

```php
<?php
use Bitrix\Main\ORM\Fields\Relations\Reference;
use Bitrix\Main\ORM\Query\Join;
use Bitrix\Sale\Internals\OrderPropsValueTable;
use Bitrix\Sale\Internals\OrderTable;

$rows = OrderTable::getList([
    'select' => ['ID', 'DATE_INSERT', 'PROP_VALUE.VALUE'],
    'filter' => ['!STATUS_ID' => ['C', 'RS']],
    'runtime' => [
        new Reference(
            'PROP_VALUE',
            OrderPropsValueTable::class,
            Join::on('this.ID', 'ref.ORDER_ID')
                ->where('ref.CODE', '=', 'LOCATION')
        ),
    ],
])->fetchAll();
```

### Что часто ломают

1. Не ограничивают JOIN дополнительными условиями.
2. Делают `LEFT`, хотя нужен `INNER`.
3. Тащат поля из join-таблиц без необходимости.

### Что правильно

- Сначала проектируем SQL-модель, потом пишем ORM-обёртку.
- Проверяем generated SQL и `EXPLAIN`.

---

## 4) Неправильный `JOIN`-тип

`LEFT JOIN` по умолчанию удобен, но не всегда эффективен.

- `INNER JOIN`: когда нужны только совпавшие строки.
- `LEFT JOIN`: когда связанные данные опциональны.

```php
<?php
use Bitrix\Main\ORM\Fields\Relations\Reference;
use Bitrix\Main\ORM\Query\Join;

$ref = (new Reference(
    'FILE',
    \Bitrix\Main\FileTable::class,
    Join::on('this.FILE_ID', 'ref.ID')
))->configureJoinType(Join::TYPE_INNER);
```

Проверка всегда одна: смотрим план выполнения, а не спорим по привычкам.

---

## 5) Индексы «по интуиции» вместо индексов по профилю нагрузки

ORM-запросы ускоряются индексами точно так же, как ручной SQL.

### Нормальный порядок

1. Снять slow queries.
2. Прогнать `EXPLAIN`.
3. Добавить индекс под конкретный фильтр/джойн.
4. Перепроверить план и фактическое время.

```sql
CREATE INDEX ix_ibep_prop_valnum
ON b_iblock_element_property (IBLOCK_PROPERTY_ID, VALUE_NUM);
```

ВНИМАНИЕ: DDL-изменения только после теста на копии базы и согласования с DBA.

---

## 6) Кэш без политики инвалидации

`cache` в ORM это полезно, но без инвалидации это источник устаревших данных.

```php
<?php
$rows = \Bitrix\Iblock\ElementTable::getList([
    'select' => ['ID', 'NAME'],
    'filter' => ['=IBLOCK_ID' => 7, '=ACTIVE' => 'Y'],
    'cache' => [
        'ttl' => 1800,
        'cache_joins' => true,
    ],
])->fetchAll();
```

Профессиональный подход:

1. Документируем, какие бизнес-события сбрасывают кэш.
2. Разделяем read-heavy и write-heavy сценарии.
3. На этапе диагностики отключаем кэш и смотрим «чистую» стоимость.

---

## Часть 2. Практика Query API, которую редко показывают в статьях

## 7) Агрегации через `ExpressionField` и HAVING-логика

```php
<?php
use Bitrix\Main\Entity\ExpressionField;
use Bitrix\Iblock\ElementTable;

$query = ElementTable::query()
    ->registerRuntimeField(new ExpressionField('CNT', 'COUNT(%s)', ['NAME']))
    ->setSelect(['NAME', 'CNT'])
    ->setFilter(['=IBLOCK_ID' => 7, '>CNT' => 5])
    ->setGroup(['NAME']);

$rows = $query->exec()->fetchAll();
```

Здесь фильтр по `CNT` переносится в `HAVING`, и это мощно для аналитических срезов.

---

## 8) Сложная логика условий через `ConditionTree`

```php
<?php
use Bitrix\Main\ORM\Query\Query;
use Bitrix\Iblock\ElementTable;

$rows = ElementTable::query()
    ->setSelect(['ID', 'NAME'])
    ->where(
        Query::filter()
            ->logic('or')
            ->where('ACTIVE', '=', 'Y')
            ->where('SORT', '>', 500)
    )
    ->setLimit(50)
    ->exec()
    ->fetchAll();
```

Плюс: читаемая логика без вложенного «супа» в массивах фильтра.

---

## 9) Кастомные условия в JOIN через `Join::on()->where()`

```php
<?php
use Bitrix\Main\ORM\Fields\Relations\Reference;
use Bitrix\Main\ORM\Query\Join;

$reference = new Reference(
    'FILE',
    \Bitrix\Main\FileTable::class,
    Join::on('this.FILE_ID', 'ref.ID')
        ->where('ref.WIDTH', '<=', 1600)
        ->where('ref.HEIGHT', '<=', 1600)
);
```

Это особенно полезно, когда условие должно быть именно в `JOIN`, а не в общем `WHERE`.

---

## 10) ORM-классы инфоблока через API-код: что нужно помнить

Для полноценной ORM-работы с элементами инфоблока часто используют класс вида:

- `\Bitrix\Iblock\Elements\Element<API_CODE>Table`

Это даёт более естественную работу со свойствами и объектной моделью (`fetchObject`, `fetchCollection`).

Ключевая оговорка: не нужно тащить всю объектную магию в high-load чтение витрин.  
Для тяжёлых списков часто выгоднее плоский `fetchAll()` + явная сборка DTO/read-model.

---

## Часть 3. Диагностика и контроль регрессий

## 11) Что обязательно смотреть после каждого «ускорения»

1. Количество SQL-запросов.
2. Топ медленных SQL.
3. Изменение `P95/P99`.
4. Пиковую память.
5. Корректность бизнес-результата (не только скорость).

```php
<?php
\Bitrix\Main\Diag\SqlTracker::getInstance()->start();

// ... ORM код

$queries = \Bitrix\Main\Diag\SqlTracker::getInstance()->getQueries();
```

---

## 12) Пример боевого отчёта «до/после»

| Метрика | До | После |
|---|---:|---:|
| SQL-запросы | 312 | 74 |
| P95 server time, ms | 2260 | 690 |
| Peak memory, MB | 34 | 16 |
| Внешние API вызовы | 11 | 4 |

Что изменили:

- убрали N+1 в каталоге;
- сократили `select` и runtime поля;
- перепроектировали `LEFT JOIN` -> `INNER JOIN` в критичном месте;
- добавили индекс под основной фильтр;
- убрали лишнее кэширование, оставили осмысленное.

---

## 13) Чек-лист для тимлида перед merge

1. Запрос решает задачу минимально достаточными полями.
2. Нет скрытых N+1 по ценам, свойствам, разделам, файлам.
3. Runtime relation обоснован и проверен по SQL/EXPLAIN.
4. JOIN-тип выбран по плану, а не по привычке.
5. Под ключевые фильтры есть индексы или план по их добавлению.
6. Кэш-стратегия и инвалидация описаны.
7. Есть измерения «до/после» и проверка бизнес-корректности.

---

## Частые вопросы из практики

## ORM всегда быстрее старого API?

Нет. Быстрее бывает хорошо спроектированный запрос и корректная стратегия данных.

## Нужно ли полностью переписывать legacy на D7 ORM?

Нет. Обычно выгоднее эволюционный путь: переписывать горячие участки, где есть measurable pain.

## Когда уходить в чистый SQL?

Когда ORM начинает мешать выразить эффективный запрос или существенно ухудшает план исполнения. Но и там оставляем стандарты: читаемость, тестируемость, измеримость.

---

## Финальный вывод

D7 ORM в 1C-Битрикс это сильный инструмент для production, если использовать его инженерно:

- проектировать запросы как SQL-модель,
- осознанно строить runtime relation и JOIN,
- работать с индексами на фактах,
- измерять эффект и контролировать регрессии.

Тогда ORM становится не декоративным слоем, а реальным усилителем скорости разработки и стабильности проекта.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
