# Оптимизация производительности 1C-Битрикс: 10 частых ошибок и быстрые исправления
### Практический разбор с кодом, антипаттернами, метриками и инструментами диагностики

> Формат: проблема -> почему больно -> как исправить -> чем проверить  
> Цель: ускорить Bitrix-проект системно, без рефакторинга «ради рефакторинга»  
> Для кого: junior/middle/senior разработчики, техлиды, архитекторы

---

## Введение

Когда проект на 1C-Битрикс начинает тормозить, команда обычно делает две ошибки:

1. Оптимизирует «на глаз».
2. Чинит симптомы, а не причину.

Результат предсказуем: времени потрачено много, ускорение минимальное.

Главный принцип этой статьи: сначала измеряем, потом меняем код, потом повторно измеряем.

---

## 1) N+1 запросов в циклах

### Типичный антипаттерн

```php
<?php
$res = CIBlockElement::GetList([], ['IBLOCK_ID' => 2], false, false, ['ID', 'NAME']);
while ($el = $res->Fetch()) {
    $price = CPrice::GetBasePrice($el['ID']);
    $props = CIBlockElement::GetProperty(2, $el['ID']);
}
```

### Почему это больно

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

### Инструменты поиска

- `\Bitrix\Main\Diag\SqlTracker`
- Монитор производительности Bitrix
- Повторяющиеся SQL с одинаковыми параметрами

---

## 2) «Берём всё» из инфоблока

### Антипаттерн

```php
<?php
CIBlockElement::GetList([], $filter, false, false, []);
// или
CIBlockElement::GetList([], $filter, false, false, ['*']);
```

### Почему это больно

Тащатся поля, которые не используются (`DETAIL_TEXT`, `TAGS` и т.д.), растёт сеть между БД и PHP, память и время.

### Как исправить

```php
<?php
$select = ['ID', 'NAME', 'CODE', 'IBLOCK_SECTION_ID', 'ACTIVE_FROM'];
$res = CIBlockElement::GetList([], $filter, false, false, $select);
```

Если нужны свойства у набора элементов, не вызываем `GetProperty` в цикле:

```php
<?php
\CIBlockElement::GetPropertyValuesArray($props, $iblockId, $idFilter);
```

---

## 3) Повторные вызовы `CIBlockElement::GetList` для одной логики

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

Простой вариант для компонента: `static $elementsCache = [];`.

---

## 4) Бизнес-логика в `template.php`

### Антипаттерн

Шаблон делает SQL, HTTP-запросы и расчёты.

### Как правильно

1. `result_modifier.php` — подготовка данных.
2. `template.php` — только отображение.
3. Сложная логика — в сервисах/хендлерах.

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

## 5) Нет кэширования там, где оно необходимо

### Базовая настройка компонента

```php
<?php
$arParams['CACHE_TYPE'] = 'A';
$arParams['CACHE_TIME'] = 3600;
```

### Кэш для своих выборок

```php
<?php
$cacheId = 'my_cache_' . md5(serialize($filter));
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

### Тегированная инвалидация

```php
<?php
use Bitrix\Main\Application;

$taggedCache = Application::getInstance()->getTaggedCache();
$taggedCache->startTagCache($cachePath);
$taggedCache->registerTag('iblock_id_' . $iblockId);
$taggedCache->endTagCache();
```

---

## 6) Неправильная пагинация

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

Ориентир для фронта: обычно 20-50 элементов достаточно.

---

## 7) Повторные запросы во внешние API

### Антипаттерн

Один и тот же URL вызывается много раз за один hit.

### Исправление

```php
<?php
static $apiCache = [];
if (!isset($apiCache[$url])) {
    $apiCache[$url] = file_get_contents($url);
}
$data = $apiCache[$url];
```

Для более тяжёлых сценариев: `CPHPCache` + очередь через агенты.

```php
<?php
\CAgent::AddAgent('MyIntegration::processQueue();', 'my_module', 'N', 60);
```

---

## 8) Слишком много обработчиков событий

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

## 9) Нет индексов под реальные фильтры

### Симптом

Фильтры по свойствам работают медленно на больших объёмах.

### Что проверить

- slow query log MySQL;
- монитор производительности Bitrix;
- планы запросов (`EXPLAIN`).

### Пример индекса (только после проверки на стенде)

```sql
CREATE INDEX ix_iproperty_123_num
ON b_iblock_element_property(IBLOCK_PROPERTY_ID, VALUE_NUM);
```

ВНИМАНИЕ: индексы нужно согласовывать с DBA и обязательно проверять на копии базы.

---

## 10) Оптимизация без профилирования

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

## Вывод

Быстрый Bitrix-проект почти всегда строится на дисциплине:

- точные выборки,
- контроль числа запросов,
- корректное кэширование,
- профилирование до и после правок.

Без этого «оптимизация» превращается в лотерею.

---

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
