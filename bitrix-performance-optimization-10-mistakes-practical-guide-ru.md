# Производительность 1C-Битрикс на практике: 10 системных ошибок и как их устранять
### Разбор для техлида: причинно-следственные связи, корректные паттерны, код и проверка результата

> Формат: симптом -> техническая причина -> инженерное решение -> как проверить эффект  
> Цель: уменьшить latency и cost of change без архитектурных компромиссов  
> Для кого: middle/senior разработчики, техлиды, архитекторы

---

## Введение: о чём на самом деле спорят, когда говорят «у нас тормозит Bitrix»

На production «тормозит» почти никогда не одна проблема. Обычно это комбинация:

1. Избыточной нагрузки на БД (много запросов, плохие планы, отсутствие индексов).
2. Неправильной организации чтения данных в PHP (N+1, дубли выборок, лишние преобразования).
3. Неконтролируемых внешних зависимостей (HTTP-интеграции без дедупликации и таймаутов).
4. Несогласованного кэширования (кэш есть, но политика инвалидации не соответствует бизнес-событиям).

Практический принцип один: фиксируем baseline, меняем одно узкое место, повторно измеряем. Так мы понимаем, что сработало, а что было шумом.

---

## Базовая модель диагностики (перед любым рефакторингом)

Перед правками фиксируем 4 метрики на одной и той же странице/сценарии:

1. `TTFB / server time`.
2. Количество SQL-запросов и топ-5 по времени.
3. Количество и длительность внешних HTTP-вызовов.
4. Peak memory.

Без этой базы «оптимизация» быстро превращается в субъективную оценку.

---

## 1) N+1 в чтении каталога и свойств

### Симптом

При открытии раздела каталога число SQL-запросов растёт линейно от числа элементов.

### Причина

Внутри цикла по элементам выполняются отдельные обращения за ценами/свойствами.

### Антипаттерн (legacy-стиль)

```php
<?php
// На каждом элементе отдельные обращения к БД
$items = [];
$res = CIBlockElement::GetList([], ['IBLOCK_ID' => $iblockId, 'ACTIVE' => 'Y'], false, ['nTopCount' => 100], ['ID', 'NAME']);
while ($item = $res->Fetch()) {
    $items[] = $item;
    $price = CPrice::GetBasePrice((int)$item['ID']);
}
```

### Корректный подход

В D7 собираем ID один раз и дочитываем связанные данные пакетно через ORM.

```php
<?php
use Bitrix\Catalog\PriceTable;
use Bitrix\Iblock\ElementTable;
use Bitrix\Iblock\ElementPropertyTable;

$elements = ElementTable::getList([
    'select' => ['ID', 'NAME', 'IBLOCK_ID'],
    'filter' => ['=IBLOCK_ID' => $iblockId, '=ACTIVE' => 'Y'],
    'order' => ['ID' => 'DESC'],
    'limit' => 100,
])->fetchAll();

$elementIds = array_map(static fn(array $row): int => (int)$row['ID'], $elements);

$priceRows = PriceTable::getList([
    'select' => ['PRODUCT_ID', 'PRICE'],
    'filter' => ['@PRODUCT_ID' => $elementIds, '=CATALOG_GROUP_ID' => 1],
])->fetchAll();

$priceMap = [];
foreach ($priceRows as $row) {
    $priceMap[(int)$row['PRODUCT_ID']] = (float)$row['PRICE'];
}

$propertyRows = ElementPropertyTable::getList([
    'select' => ['IBLOCK_ELEMENT_ID', 'IBLOCK_PROPERTY_ID', 'VALUE'],
    'filter' => ['@IBLOCK_ELEMENT_ID' => $elementIds],
])->fetchAll();
```

### Проверка

Сравнить `query count` и P95 server time до/после.

---

## 2) Избыточная выборка полей и свойств

### Симптом

Высокая память и время сериализации даже при относительно небольшом числе элементов.

### Причина

Используются `['*']` или пустой `select`, а в шаблоне реально нужны 5-7 полей.

### Корректный подход

Формировать `select` только из реально используемых полей. Для свойств — отдельная осознанная стратегия чтения.

```php
<?php
use Bitrix\Iblock\ElementTable;

$rows = ElementTable::getList([
    'select' => ['ID', 'IBLOCK_ID', 'NAME', 'CODE', 'PREVIEW_PICTURE'],
    'filter' => ['=IBLOCK_ID' => $iblockId, '=ACTIVE' => 'Y'],
    'order' => ['SORT' => 'ASC', 'ID' => 'DESC'],
    'limit' => 30,
])->fetchAll();
```

### Проверка

Измерить уменьшение memory peak и времени выполнения выборки.

---

## 3) Повтор одинаковых запросов в одном HTTP-хите

### Симптом

Один и тот же SQL повторяется несколько раз с одинаковыми параметрами.

### Причина

Несколько слоёв (controller, service, view-model mapper) повторно вызывают одинаковую выборку.

### Корректный подход

Вводим request-scope cache (локально на время хита), но не путаем его с постоянным кэшем.

```php
<?php
final class ProductReadModel
{
    private array $requestCache = [];

    public function getBySection(int $sectionId): array
    {
        $key = 'section:' . $sectionId;
        if (!array_key_exists($key, $this->requestCache)) {
            $this->requestCache[$key] = $this->loadBySection($sectionId);
        }

        return $this->requestCache[$key];
    }

    private function loadBySection(int $sectionId): array
    {
        // Один запрос в БД
        return [];
    }
}
```

---

## 4) Бизнес-правила в слое представления

### Симптом

Шаблон содержит SQL/HTTP/расчёты, ревью и изменение логики становятся дорогими.

### Причина

Смешение Presentation и Application слоёв.

### Корректный подход

1. Application/service слой готовит DTO/read model.
2. Presentation-слой только маппит данные под экран.
3. Шаблон/контроллер не ходит в БД и внешние API.

```php
<?php
// Presentation mapper
/** @var array $arResult */
$arResult['CARD_VIEW_MODELS'] = array_map(
    static fn(array $item): array => [
        'TITLE' => (string)$item['NAME'],
        'URL' => (string)$item['DETAIL_PAGE_URL'],
        'PRICE' => number_format((float)$item['PRICE'], 0, '.', ' '),
    ],
    $arResult['ITEMS']
);
```

---

## 5) Кэш без политики инвалидации

### Симптом

После обновления контента пользователи видят устаревшие данные.

### Причина

Кэш включён, но не связан с бизнес-событиями изменения данных.

### Корректный подход

- Для чтения данных используем `\Bitrix\Main\Data\Cache`.
- Для согласованности с инфоблоками — tagged cache.
- Ключ кэша включает только значимые параметры (не весь request).

```php
<?php
use Bitrix\Main\Application;
use Bitrix\Main\Data\Cache;

$cacheTtl = 1800;
$cachePath = '/catalog/list/';
$cacheId = 'catalog:' . md5(implode('|', [$iblockId, $sectionId, $page, $sortField, $sortOrder]));
$cache = Cache::createInstance();

if ($cache->InitCache($cacheTtl, $cacheId, $cachePath)) {
    $result = $cache->GetVars();
} elseif ($cache->StartDataCache()) {
    $tagged = Application::getInstance()->getTaggedCache();
    $tagged->startTagCache($cachePath);
    $tagged->registerTag('iblock_id_' . $iblockId);

    $result = []; // загрузка данных

    $tagged->endTagCache();
    $cache->EndDataCache($result);
}
```

### Важно

Кэширование без корректной инвалидации даёт не ускорение, а недостоверные данные.

---

## 6) Пагинация без ограничений и стабильной сортировки

### Симптом

Страница раздела медленная, а состав элементов «прыгает» между страницами.

### Причина

Большой page size и/или сортировка без стабильного вторичного поля.

### Корректный подход

```php
<?php
use Bitrix\Iblock\ElementTable;
use Bitrix\Main\ORM\Query\Query;

$query = ElementTable::query()
    ->setSelect(['ID', 'NAME', 'SORT'])
    ->setFilter(['=IBLOCK_ID' => $iblockId, '=ACTIVE' => 'Y'])
    ->setOrder(['SORT' => 'ASC', 'ID' => 'DESC'])
    ->setOffset(($page - 1) * 30)
    ->setLimit(30);

$rows = $query->fetchAll();
```

---

## 7) HTTP-интеграции без контроля отказов

### Симптом

Внешний API «подвешивает» рендер страницы.

### Причина

Нет таймаутов, ретраев, дедупликации и fallback-стратегии.

### Корректный подход

```php
<?php
$http = new \Bitrix\Main\Web\HttpClient([
    'socketTimeout' => 2,
    'streamTimeout' => 3,
    'disableSslVerification' => false,
]);

$response = $http->get($url);
$status = $http->getStatus();

if ($status !== 200 || $response === false) {
    // fallback: кэшированное значение / graceful degradation
}
```

Для тяжёлых задач (массовая синхронизация) переводим обработку в фон: cron/очереди/воркеры. Агенты в Bitrix подойдут только для лёгких и контролируемых задач.

---

## 8) Избыточные обработчики событий

### Симптом

Обновление элемента или заказа выполняется медленно даже без сложного UI.

### Причина

Тяжёлая логика подвешена на «горячие» события (`OnBefore...`, `OnAfter...`) без контроля стоимости.

### Корректный подход

- Проводить аудит обработчиков раз в релиз.
- Убирать side effects из `OnBefore*`, оставляя минимальную валидацию.
- Тяжёлые операции выносить в фоновые задачи.

```bash
rg "AddEventHandler|EventManager::getInstance\(\)->addEventHandler" /path/to/project
```

---

## 9) Индексы «на глаз» вместо анализа планов

### Симптом

Есть фильтры по свойствам, но БД всё равно сканирует большие объёмы.

### Причина

Индексы отсутствуют или не соответствуют реальным WHERE/ORDER BY условиям.

### Корректный подход

1. Собрать медленные запросы.
2. Прогнать `EXPLAIN`.
3. Добавить индекс под конкретный паттерн запроса.
4. Повторно проверить план и фактическое время.

```sql
CREATE INDEX ix_ibep_prop_valnum
ON b_iblock_element_property (IBLOCK_PROPERTY_ID, VALUE_NUM);
```

ВНИМАНИЕ: любые DDL-изменения только после проверки на копии БД и согласования с DBA.

---

## 10) Нет baseline и regression-контроля

### Симптом

После «оптимизации» непонятно, стало ли реально быстрее и не сломали ли соседний сценарий.

### Корректный подход

- Фиксируем baseline-метрики до правок.
- Вносим изменения порционно.
- Снимаем те же метрики после.
- Храним результат в коротком perf-отчёте (таблица/markdown в репозитории).

Инструменты:

1. Bitrix Performance Monitor.
2. `\Bitrix\Main\Diag\SqlTracker`.
3. Xdebug/Blackfire/New Relic/Datadog (в зависимости от контура).

---

## Пример отчёта «до/после»

| Метрика | До | После |
|---|---:|---:|
| SQL-запросы | 328 | 47 |
| P95 server time, ms | 2150 | 412 |
| Peak memory, MB | 32 | 12 |
| Внешние HTTP-вызовы | 14 | 3 |

---

## Финальный чек-лист для техлид-ревью

1. Есть baseline и методика замера.
2. Оптимизация адресует конкретный bottleneck, а не «всё сразу».
3. Изменения ограничены по зоне ответственности.
4. Кэш синхронизирован с политикой инвалидации.
5. Есть проверка на регрессии по функциональности и данным.

---

## Вывод

Производительность в Bitrix это не история про «ускорить любой ценой».  
Это спокойная инженерная рутина: измерили, устранили узкое место, подтвердили результат цифрами.  
Так команда не спорит «по ощущениям», а движется по фактам.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
