# Почему тормозит 1C-Битрикс: 10 ошибок, которые съедают производительность
### Практическое руководство для code review: теория, антипаттерны, код и метрики «до/после»

> Формат: проблема -> причина -> корректное решение -> верификация  
> Цель: ускорить Bitrix-проект системно, без рефакторинга «ради рефакторинга»  
> Для кого: junior/middle/senior разработчики, техлиды, архитекторы

---

## Введение: где обычно теряется производительность в Bitrix

Когда проект на 1C-Битрикс начинает тормозить, команда обычно делает две ошибки:

1. Оптимизирует «на глаз».
2. Чинит симптомы, а не причину.

Результат предсказуем: много работы, слабый эффект и регрессии в соседних местах.

Главный принцип этой статьи: сначала измеряем, потом меняем код, потом повторно измеряем.

Если коротко, производительность в Bitrix почти всегда проседает в четырех зонах:

1. SQL (слишком много запросов или плохие планы).
2. PHP-слой (лишняя работа в циклах и шаблонах).
3. Внешние интеграции (блокирующие HTTP-вызовы).
4. Кэш (либо его нет, либо он настроен так, что только мешает).

Дальше разберём 10 ошибок, которые чаще всего всплывают на production-проектах.

---

## 1) N+1 в циклах: из локального удобства в системную деградацию

### Типичный антипаттерн

```php
<?php
$res = CIBlockElement::GetList([], ['IBLOCK_ID' => 2], false, false, ['ID', 'NAME']);
while ($el = $res->Fetch()) {
    $price = CPrice::GetBasePrice((int)$el['ID']);
    $props = CIBlockElement::GetProperty(2, (int)$el['ID']);
}
```

### Почему это больно

На малом объёме это почти не видно, но при росте каталога резко увеличиваются задержки БД и время TTFB.
100 элементов легко превращаются в 200+ SQL-запросов.

### Как исправить

Собираем ID, затем одним запросом получаем цены:

```php
<?php
$ids = [];
$res = CIBlockElement::GetList([], ['IBLOCK_ID' => 2], false, false, ['ID', 'NAME']);
while ($el = $res->Fetch()) {
    $ids[] = (int)$el['ID'];
}

$prices = [];
$dbPrice = CPrice::GetList([], ['PRODUCT_ID' => $ids]);
while ($price = $dbPrice->Fetch()) {
    $prices[(int)$price['PRODUCT_ID']] = (float)$price['PRICE'];
}
```

### Как диагностировать

- `\Bitrix\Main\Diag\SqlTracker`
- Монитор производительности Bitrix
- Повторяющиеся SQL с одинаковыми параметрами

---

## 2) Избыточная выборка полей: лишняя нагрузка на БД, сеть и PHP

### Антипаттерн

```php
<?php
CIBlockElement::GetList([], $filter, false, false, []);
// или
CIBlockElement::GetList([], $filter, false, false, ['*']);
```

### Почему это больно

Тащатся поля, которые не используются (`DETAIL_TEXT`, `TAGS` и т.д.). В итоге растёт объём данных между БД и PHP, увеличиваются память и время рендера.

### Как исправить

```php
<?php
$select = ['ID', 'NAME', 'CODE', 'IBLOCK_SECTION_ID', 'ACTIVE_FROM'];
$res = CIBlockElement::GetList([], $filter, false, false, $select);
```

Если нужны свойства для набора элементов, не вызываем `GetProperty` в цикле:

```php
<?php
\CIBlockElement::GetPropertyValuesArray(
    $props,
    $iblockId,
    ['ID' => $ids]
);
```

---

## 3) Повторяем одинаковые запросы в рамках одного HTTP-запроса

### Антипаттерн

Одинаковый фильтр вызывается несколько раз в рамках одного запроса.

### Как исправить

```php
<?php
final class ElementRepository
{
    private static array $cache = [];

    public static function getBySection(int $sectionId): array
    {
        $key = 'section_' . $sectionId;
        if (!isset(self::$cache[$key])) {
            self::$cache[$key] = []; // здесь выборка GetList
        }

        return self::$cache[$key];
    }
}
```

Простой и безопасный вариант для компонента: `static $elementsCache = [];` только на время текущего запроса.

---

## 4) Бизнес-логика в `template.php`: рост связности и сложности поддержки

### Антипаттерн

Шаблон делает SQL, HTTP-запросы и расчёты.

### Немного теории

Шаблон в Bitrix отвечает за представление, а не за бизнес-решения и IO-операции.
Чем «умнее» становится `template.php`, тем тяжелее поддержка и выше риск регрессий.

### Как правильно

1. `component.php`/сервис — загрузка и бизнес-обработка данных.
2. `result_modifier.php` — подготовка к отображению.
3. `template.php` — только отображение.
3. Сложная логика — в сервисах/хендлерах (Application/Domain слой).

```php
<?php
// result_modifier.php
$arResult['PROCESSED_ITEMS'] = MyBusinessService::enrichItems($arResult['ITEMS']);
```

```php
<?php
// template.php
foreach ($arResult['PROCESSED_ITEMS'] as $item) {
    echo $item['DISPLAY_PRICE'];
}
```

---

## 5) Отсутствие осмысленного кэширования

### Базовая настройка компонента

```php
<?php
$arParams['CACHE_TYPE'] = 'A';
$arParams['CACHE_TIME'] = 3600;
```

### Кэш для своих выборок

```php
<?php
$cacheId = 'my_cache_' . md5(json_encode($filter, JSON_UNESCAPED_UNICODE));
$cachePath = '/my_component/';
$cache = new CPHPCache();

if ($cache->InitCache(3600, $cacheId, $cachePath)) {
    $arResult = $cache->GetVars();
} else {
    $arResult = []; // запросы
    $cache->StartDataCache();
    $cache->EndDataCache($arResult);
}
```

### Тегированная инвалидация (предпочтительно для данных инфоблоков)

```php
<?php
use Bitrix\Main\Application;

$taggedCache = Application::getInstance()->getTaggedCache();
$taggedCache->startTagCache($cachePath);
$taggedCache->registerTag('iblock_id_' . $iblockId);
$taggedCache->endTagCache();
```

ВНИМАНИЕ: кэш без корректной инвалидации создаёт «призрачные» баги и устаревшие данные.

---

## 6) Пагинация без ограничений: грузим лишнее и на сервер, и в браузер

### Проблема

На страницу грузится слишком много элементов.

### Решение

```php
<?php
$nav = new \Bitrix\Main\UI\PageNavigation('nav');
$nav->setPageSize(20)->initFromUri();

$res = CIBlockElement::GetList(
    [],
    $filter,
    false,
    ['nPageSize' => $nav->getPageSize(), 'iNumPage' => $nav->getCurrentPage()],
    $select
);
$nav->setRecordCount($res->NavRecordCount);
```

Практический ориентир для витрины: 20-50 элементов на страницу. Всё, что выше, обычно требует отдельного UX-сценария.

---

## 7) Внешние API без дедупликации и таймаутов

### Антипаттерн

Один и тот же URL вызывается повторно в рамках одного HTTP-запроса.

### Исправление

```php
<?php
static $apiCache = [];
if (!isset($apiCache[$url])) {
    $httpClient = new \Bitrix\Main\Web\HttpClient([
        'socketTimeout' => 2,
        'streamTimeout' => 3,
    ]);
    $apiCache[$url] = $httpClient->get($url);
}
$data = $apiCache[$url];
```

Для production обязательно контролировать таймауты, обработку не-200 ответов и ретраи. Для тяжёлых сценариев — `CPHPCache` + очередь через агенты/воркеры.

```php
<?php
\CAgent::AddAgent('MyIntegration::processQueue();', 'my_module', 'N', 60);
```

---

## 8) Неконтролируемый рост обработчиков событий

### Где искать

- `/bitrix/php_interface/init.php`
- `/local/modules/*`
- `/local/*_event.php`

### Почему тормозит

Тяжёлая логика в горячих событиях вроде `OnBeforeIBlockElementUpdate`.

### Что делать

- убирать лишние обработчики;
- выносить тяжёлую работу в фон;
- периодически аудировать регистрацию обработчиков:

```bash
rg "AddEventHandler|EventManager::getInstance\\(\\)->addEventHandler" /path/to/project
```

---

## 9) Фильтрация по свойствам без проверки планов запросов

### Симптом

Фильтры по свойствам работают медленно на больших объёмах.

### Что проверить

- slow query log MySQL;
- монитор производительности Bitrix;
- планы запросов (`EXPLAIN`).

### Пример индекса (только после проверки на копии и `EXPLAIN`)

```sql
CREATE INDEX ix_ibep_prop_valnum
ON b_iblock_element_property (IBLOCK_PROPERTY_ID, VALUE_NUM);
```

ВНИМАНИЕ: индексы согласовываются с DBA, проверяются на копии базы и валидируются по фактическому профилю нагрузки.

---

## 10) Оптимизация без базовой телеметрии

### Инструменты

1. Монитор производительности Bitrix.
2. `SqlTracker`:

```php
<?php
\Bitrix\Main\Diag\SqlTracker::getInstance()->start();
// ваш код
$queries = \Bitrix\Main\Diag\SqlTracker::getInstance()->getQueries();
```

3. Xdebug + Webgrind (или аналог) для PHP-профилирования.
4. APM (New Relic, Datadog, Blackfire) для боевого профиля, если доступно.

### Мини-чеклист замера

- время генерации страницы;
- число SQL-запросов;
- доля времени на внешние API;
- пиковая память.

---

## Метрики «до/после» (пример)

| Метрика | До | После |
|---------|----|--------|
| SQL-запросы | 328 | 47 |
| Время (ms) | 2150 | 412 |
| Память (MB) | 32 | 12 |

---

## Финальный чек-лист перед релизом оптимизации

1. Замеры «до» зафиксированы.
2. Изменения внесены точечно, без слома архитектуры.
3. Замеры «после» подтверждают эффект.
4. Кэш и инвалидация проверены.
5. Нет побочных деградаций в функциональности.

---

## Вывод: производительность — это инженерная дисциплина, а не набор трюков

Быстрый Bitrix-проект почти всегда строится на дисциплине:

- точные выборки,
- контроль числа запросов,
- корректное кэширование,
- профилирование до и после правок.

Без этого оптимизация превращается в лотерею: усилий много, результата мало.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
