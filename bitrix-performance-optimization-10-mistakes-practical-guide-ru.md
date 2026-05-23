# Почему тормозит 1C-Битрикс: 10 ошибок, которые съедают производительность
### Живое практическое руководство: теория, антипаттерны, код и метрики «до/после»

> Формат: проблема -> почему больно -> как исправить -> чем проверить  
> Цель: ускорить Bitrix-проект системно, без рефакторинга «ради рефакторинга»  
> Для кого: junior/middle/senior разработчики, техлиды, архитекторы

---

## Введение: где на самом деле умирает скорость в Bitrix

Когда проект на 1C-Битрикс начинает тормозить, команда обычно делает две ошибки:

1. Оптимизирует «на глаз».
2. Чинит симптомы, а не причину.

Результат предсказуем: времени потрачено много, ускорение минимальное.

Главный принцип этой статьи: сначала измеряем, потом меняем код, потом повторно измеряем.

Если коротко, производительность в Bitrix почти всегда проседает в четырех зонах:

1. SQL (слишком много запросов или плохие планы).
2. PHP-слой (лишняя работа в циклах и шаблонах).
3. Внешние интеграции (блокирующие HTTP-вызовы).
4. Кэш (либо его нет, либо он настроен так, что только мешает).

Дальше пройдемся по 10 ошибкам, которые встречаются чаще всего на боевых проектах.

---

## 1) N+1 в циклах: классика, которая незаметно убивает страницу

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

На маленьких данных это может быть незаметно, но на каталоге в 5-10 тысяч элементов сервер начинает «задыхаться».
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

## 2) Выборка «на всякий случай»: берём всё, используем 20%

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

Если нужны свойства у набора элементов, не вызываем `GetProperty` в цикле:

```php
<?php
\CIBlockElement::GetPropertyValuesArray($props, $iblockId, $idFilter);
```

---

## 3) Дублируем один и тот же `GetList`: платим за один ответ несколько раз

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

## 4) `template.php` как «центр вселенной»: дорого и хрупко

### Антипаттерн

Шаблон делает SQL, HTTP-запросы и расчёты.

### Немного теории

Шаблон в Bitrix должен отвечать за представление, а не за бизнес-решения.
Чем «умнее» становится `template.php`, тем тяжелее поддержка и выше риск регрессий.

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

## 5) Игнорируем кэш там, где данные почти статичны

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

Ориентир для фронта: обычно 20-50 элементов достаточно.

---

## 7) Внешние API в лоб: один и тот же вызов по кругу

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

## 8) Событийная «лапша»: когда обработчиков много, а контроля мало

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

## 9) Фильтруем по свойствам без индексов

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

## 10) Оптимизация «на ощущениях» вместо профилирования

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

## Вывод: производительность это не магия, а инженерная дисциплина

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
