# Агенты, cron и очереди в 1C-Битрикс: production-архитектура надежной фоновой обработки
### Практическое руководство по SLA, идемпотентности, retry/backoff и управляемой эксплуатации

> Формат: боль -> причина -> решение -> код -> эксплуатация  
> Для кого: middle/senior разработчики, техлиды, архитекторы, технические владельцы Bitrix-проектов

---

## Системная причина инцидентов в фоновой обработке Bitrix

Сценарий повторяется от проекта к проекту: сначала агент в `init.php` кажется удобным, но с ростом нагрузки появляется хвост задач, а после релизов и пиков трафика фоновые операции начинают отставать от SLA. Дальше цепочка предсказуема: тормозит портал, отваливаются интеграции, бизнес видит проблему раньше команды.  
Корневая причина обычно не в платформе, а в архитектуре запуска и отсутствии эксплуатационного стандарта.

---

## Что есть в Bitrix для фоновой работы

В Bitrix есть три базовых механизма:

1. `CAgent` — регулярные задачи по расписанию.
2. `Application::addBackgroundJob()` — отложенный запуск после ответа пользователю (в рамках текущего запроса).
3. `cron` — системный запуск агентов и служебных событий вне веб-хитов.

Ключевая мысль:

1. Агент — это планировщик и оркестратор.
2. Тяжелую массовую обработку лучше делать батчами через очередь.
3. На бою агенты должны жить на `cron`, а не на случайных заходах пользователей.

---

## Теоретическая база: как мыслить фоновую обработку на уровне архитектуры

Фоновый контур — это не «выполнить код позже», а отдельная подсистема с собственными правилами надежности.

Ключевые принципы:

1. `At-least-once delivery` по умолчанию. Задача может выполниться повторно, значит обработчик обязан быть идемпотентным.
2. Разделение SLA. SLA веб-ответа и SLA фоновой обработки — разные величины, их нельзя смешивать.
3. Управление деградацией. При росте нагрузки система должна снижать throughput контролируемо, а не падать лавинообразно.
4. Наблюдаемость как часть дизайна. Метрики и алерты проектируются до запуска, а не после инцидента.

Профессиональная модель для Bitrix: `Agent (scheduler) -> Queue (state) -> Worker (execution) -> Metrics/Alerts (control loop)`.

---

## Delivery semantics простыми словами: почему дубли это норма, а не баг

В фоне есть только три практические модели доставки:

1. `At most once` — «или один раз, или потеряем». Для критичных операций обычно неприемлемо.
2. `At least once` — «минимум один раз, иногда повторно». Наиболее реалистичная модель для Bitrix.
3. `Exactly once` — дорого и сложно, в большинстве проектов имитируется через идемпотентность.

Поэтому рабочий стандарт в Bitrix всегда такой:

1. Принимаем `at least once`.
2. Убираем риск дублей через идемпотентные ключи.
3. Контролируем ошибки через retry/backoff/failed.

---

## Агенты vs background job: простая и честная матрица выбора

| Сценарий | Что выбрать | Почему |
|---|---|---|
| Регулярный импорт/синхронизация | `CAgent` + `cron` | Нужен стабильный ритм и повторяемость |
| Небольшое действие после ответа пользователю | `addBackgroundJob` | Не блокируем UX, задача короткая |
| Массовая обработка 10k+ записей | Очередь + агент-воркер | Контроль нагрузки, ретраи, статусы |
| Критичная интеграция с SLA | Очередь + `cron` + мониторинг | Нужна наблюдаемость и контроль отказов |

Если задача длинная и дорогая, нельзя рассчитывать на «просто фон».

---

## Самая частая ошибка: тяжелый агент как один огромный скрипт

Плохой подход:

1. Агент забирает весь объем данных за раз.
2. Работает 5–20 минут.
3. Падает в середине, прогресс потерян.
4. Следующий запуск накладывается, появляется хвост и лавина ошибок.

Рабочий подход:

1. Храним задачи в очереди.
2. Агент берет маленький батч (например, 20–100 записей).
3. Каждая запись обрабатывается идемпотентно.
4. Ошибки идут в retry, после лимита — в dead-letter (статус `failed`).

---

## Продакшен-правило №1: перевод агентов на cron

На хитах агенты работают нестабильно: есть трафик — есть выполнение, нет трафика — задачи «спят».

Базовая схема для прода:

1. Отключаем регулярный запуск агентов на хитах.
2. Запускаем `cron_events.php` по расписанию (обычно каждые 1–5 минут).
3. Контролируем длительность выполнения и хвост очереди.

Пример `crontab`:

```bash
    */5 * * * * /usr/bin/php -f /home/bitrix/www/bitrix/modules/main/tools/cron_events.php >> /home/bitrix/logs/cron_events.log 2>&1
```

Если задача критична по времени (например, заказы/оплаты), интервал лучше сократить до 1 минуты.

---

## Безопасная регистрация агента в модуле

```php
<?php
const MODULE_ID = 'vendor.mymodule';

CAgent::AddAgent(
    '\\Vendor\\MyModule\\Agent\\OrderSyncAgent::run();',
    MODULE_ID,
    'N',
    60,
    '',
    'Y',
    '',
    100,
    false,
    true
);
```

Почему так:

1. Код агента живет в модуле, а не в `init.php`.
2. Непериодический режим (`N`) безопаснее для длительной обработки.
3. Интервал 60 сек дает быструю реакцию без лишней агрессии к БД.

---

## Шаблон правильного агента-оркестратора

```php
<?php
namespace Vendor\MyModule\Agent;

use Vendor\MyModule\Service\QueueWorker;
use Bitrix\Main\Diag\Debug;

final class OrderSyncAgent
{
    public static function run(): string
    {
        try {
            QueueWorker::processBatch(50); // маленький контролируемый батч
        } catch (\Throwable $e) {
            Debug::writeToFile(
                [
                    'message' => $e->getMessage(),
                    'file' => $e->getFile(),
                    'line' => $e->getLine(),
                ],
                'order_sync_agent_error',
                '/local/logs/order_sync_agent.log'
            );
        }

        return '\\Vendor\\MyModule\\Agent\\OrderSyncAgent::run();';
    }
}
```

Здесь агент не «тащит мир на себе», а только запускает ограниченную порцию работы.

---

## Очередь на своей таблице: минимальный промышленный каркас

### ORM-модель

```php
<?php
namespace Vendor\MyModule\Model;

use Bitrix\Main\ORM\Data\DataManager;
use Bitrix\Main\ORM\Fields;

final class QueueTable extends DataManager
{
    public static function getTableName(): string
    {
        return 'b_vendor_queue';
    }

    public static function getMap(): array
    {
        return [
            new Fields\IntegerField('ID', ['primary' => true, 'autocomplete' => true]),
            new Fields\StringField('TYPE', ['required' => true]),
            new Fields\StringField('STATUS', ['required' => true]), // new|processing|done|retry|failed
            new Fields\TextField('PAYLOAD'),
            new Fields\IntegerField('RETRY_COUNT'),
            new Fields\DatetimeField('NEXT_RUN_AT'),
            new Fields\DatetimeField('CREATED_AT'),
            new Fields\DatetimeField('UPDATED_AT'),
            new Fields\TextField('LAST_ERROR'),
        ];
    }
}
```

### Воркер с retry и backoff

```php
<?php
namespace Vendor\MyModule\Service;

use Bitrix\Main\Application;
use Bitrix\Main\DB\SqlQueryException;
use Bitrix\Main\Type\DateTime;
use Vendor\MyModule\Model\QueueTable;

final class QueueWorker
{
    private const MAX_RETRY = 5;

    public static function processBatch(int $limit): void
    {
        $rows = self::claimBatch($limit);

        foreach ($rows as $row) {
            $id = (int)$row['ID'];
            try {
                self::handlePayload((string)$row['PAYLOAD']);
                self::markDone($id);
            } catch (\Throwable $e) {
                self::markRetry($id, (int)$row['RETRY_COUNT'], $e->getMessage());
            }
        }
    }

    /**
     * Атомарно "забирает" пачку задач в обработку.
     * Это защищает от дублей при нескольких воркерах.
     */
    private static function claimBatch(int $limit): array
    {
        $connection = Application::getConnection();
        $helper = $connection->getSqlHelper();
        $limit = max(1, (int)$limit);

        $connection->startTransaction();
        try {
            $rows = $connection->query("
                SELECT ID, PAYLOAD, RETRY_COUNT
                FROM b_vendor_queue
                WHERE STATUS IN ('new', 'retry')
                  AND (NEXT_RUN_AT IS NULL OR NEXT_RUN_AT <= NOW())
                ORDER BY ID ASC
                LIMIT {$limit}
                FOR UPDATE
            ")->fetchAll();

            if (empty($rows)) {
                $connection->commitTransaction();
                return [];
            }

            $ids = array_map(static fn(array $r): int => (int)$r['ID'], $rows);
            $idsSql = implode(',', $ids);

            $connection->queryExecute("
                UPDATE b_vendor_queue
                SET STATUS = 'processing', UPDATED_AT = NOW()
                WHERE ID IN ({$idsSql})
            ");

            $connection->commitTransaction();
            return $rows;
        } catch (\Throwable $e) {
            $connection->rollbackTransaction();
            throw $e;
        }
    }

    private static function handlePayload(string $payload): void
    {
        // Идемпотентная бизнес-операция
        // Повторный запуск не должен портить состояние.
    }

    private static function markDone(int $id): void
    {
        QueueTable::update($id, [
            'STATUS' => 'done',
            'UPDATED_AT' => new DateTime(),
        ]);
    }

    private static function markRetry(int $id, int $retryCount, string $error): void
    {
        $retryCount++;
        if ($retryCount >= self::MAX_RETRY) {
            QueueTable::update($id, [
                'STATUS' => 'failed',
                'RETRY_COUNT' => $retryCount,
                'LAST_ERROR' => $error,
                'UPDATED_AT' => new DateTime(),
            ]);
            return;
        }

        // Backoff: 1, 2, 4, 8, 16 минут
        $delayMinutes = 2 ** ($retryCount - 1);
        $nextRun = (new DateTime())->add("+{$delayMinutes} minutes");

        QueueTable::update($id, [
            'STATUS' => 'retry',
            'RETRY_COUNT' => $retryCount,
            'LAST_ERROR' => $error,
            'NEXT_RUN_AT' => $nextRun,
            'UPDATED_AT' => new DateTime(),
        ]);
    }
}
```

Это уже не «игрушечный» пример, а рабочий каркас, который переживает реальные сбои.
Ключевая часть здесь — атомарный `claimBatch()`: без него параллельные воркеры могут взять одну и ту же задачу.

---

## Реальный кейс №1: обмен заказами с CRM (до/после)

Контекст:

1. Интернет-магазин ~35k заказов/сутки.
2. Отправка событий в CRM шла напрямую из веб-хита.
3. При пике трафика время оформления заказа «плыло», часть событий терялась.

Что сделали:

1. Вынесли отправку в очередь `b_vendor_queue`.
2. Агент-воркер каждые 60 секунд, батч 40 задач.
3. Добавили `retry` до 5 попыток с экспоненциальным backoff.
4. Ввели алерт по возрасту старейшей задачи > 10 минут.

Результат за 2 недели:

1. P95 ответа checkout: 2.4s -> 1.1s.
2. Доля неотправленных событий: ~1.8% -> <0.1%.
3. Ночные инциденты по CRM-интеграции: с 4-5 в неделю до 0-1.

Почему сработало: убрали внешнюю нестабильность CRM из пользовательского запроса и сделали фон управляемым.

---

## Реальный кейс №2: массовое обновление остатков из 1С

Контекст:

1. Обновлялось до 120k SKU.
2. Один «тяжелый» агент пытался пройти все за раз.
3. При обновлении портал проседал, а хвост задач рос.

Что сделали:

1. Разбили обработку на пачки по 100 SKU.
2. Ввели приоритеты типов задач: остатки выше пересчета вторичных витрин.
3. Для конфликтующих операций добавили optimistic lock по `updated_at`.
4. Ограничили максимальное время одной итерации воркера.

Результат:

1. Средняя длительность цикла обработки: 47 мин -> 11 мин.
2. Просадка веб-производительности в пике снизилась заметно (по P95 примерно на 30–40%).
3. Количество конфликтов обновления сократилось кратно.

---

## Идемпотентность: спасает от дублей и финансовых багов

Если задача может выполниться повторно (а в фоне это нормально), операция должна быть идемпотентной.

Практически это значит:

1. Используем уникальный бизнес-ключ операции (`external_id`, `order_id+event_type`).
2. Перед записью проверяем, не обработано ли уже.
3. Не делаем необратимых действий дважды (списания, отгрузки, отправка webhook).

Без этого любой retry превращается в источник дублей.

---

## Практика идемпотентности: реальный шаблон для платежных событий

```php
<?php
use Bitrix\Main\Application;

final class PaymentEventHandler
{
    public static function handle(string $eventId, int $orderId, array $payload): void
    {
        $connection = Application::getConnection();
        $sqlHelper = $connection->getSqlHelper();

        $eventIdSql = $sqlHelper->forSql($eventId);
        $orderId = (int)$orderId;

        // ВНИМАНИЕ: на EVENT_ID должен быть UNIQUE INDEX.
        // Идемпотентность обеспечивается не проверкой SELECT, а уникальным ограничением.

        $connection->startTransaction();
        try {
            // 1) Регистрируем событие обработки.
            // Если событие уже было, INSERT упадет по unique key.
            $connection->queryExecute("
                INSERT INTO b_vendor_payment_event_log (EVENT_ID, ORDER_ID, CREATED_AT)
                VALUES ('{$eventIdSql}', {$orderId}, NOW())
            ");

            // 2) Бизнес-действие (идемпотентное изменение статуса заказа).
            self::applyOrderPaymentStatus($orderId, $payload);

            $connection->commitTransaction();
        } catch (\Throwable $e) {
            $connection->rollbackTransaction();

            // Дубликат EVENT_ID: событие уже обработано ранее.
            if (self::isDuplicateKeyError($e)) {
                return;
            }

            throw $e;
        }
    }

    private static function applyOrderPaymentStatus(int $orderId, array $payload): void
    {
        // Реализация обновления заказа.
    }

    /**
     * Определяет duplicate key по коду SQL-ошибки драйвера.
     * Для MySQL это, как правило, 1062.
     */
    private static function isDuplicateKeyError(\Throwable $e): bool
    {
        if (!$e instanceof SqlQueryException) {
            return false;
        }

        $dbCode = (int)$e->getDatabaseMessage()->getCode();
        return $dbCode === 1062;
    }
}
```

Ключевой момент: уникальный `EVENT_ID` + транзакция дают безопасную повторную доставку без дублей даже при гонках.

Минимальный DDL для этого шаблона:

```sql
CREATE UNIQUE INDEX ux_vendor_payment_event_log_event_id
    ON b_vendor_payment_event_log (EVENT_ID);
```

Если проект использует не MySQL, код duplicate key нужно брать из драйвера вашей СУБД.

---

## Multi-node и высокий SLA: когда базы и файловых блокировок уже недостаточно

На одном сервере файловые lock-механизмы и SQL-очередь обычно закрывают задачу.  
На multi-node нагрузке (несколько PHP-нод, пиковые интеграции, жесткий SLA) лучше переходить на более устойчивую схему:

1. Распределенный lock в Redis (например, ключ с TTL + compare-and-delete).
2. Внешняя очередь (RabbitMQ/Kafka) для критичных потоков с высоким объемом.
3. Отдельные воркеры по типам задач и приоритетам.
4. Dead-letter очередь как обязательный контур разбора ошибок.

Практический критерий перехода:

1. Очередь системно растет даже после тюнинга батчей и cron.
2. Часто появляются конкурирующие обработки одной задачи на разных нодах.
3. SLA по обработке бизнес-критичных событий регулярно нарушается.

---

## Redis-lock в продакшене: полноценный пример для multi-node воркеров

Ниже минимальный рабочий шаблон распределенной блокировки через Redis:

```php
<?php
namespace Vendor\MyModule\Infra;

use Redis;

final class RedisLock
{
    private Redis $redis;

    public function __construct(Redis $redis)
    {
        $this->redis = $redis;
    }

    /**
     * Пытается взять lock.
     * Возвращает token при успехе, иначе null.
     */
    public function acquire(string $key, int $ttlSeconds): ?string
    {
        $token = bin2hex(random_bytes(16));
        $ok = $this->redis->set($key, $token, ['nx', 'ex' => $ttlSeconds]);
        return $ok ? $token : null;
    }

    /**
     * Снимает lock только если token совпадает с владельцем.
     */
    public function release(string $key, string $token): bool
    {
        $lua = <<<'LUA'
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
LUA;
        return (int)$this->redis->eval($lua, [$key, $token], 1) === 1;
    }
}
```

Использование в агенте:

```php
<?php
$lock = new \Vendor\MyModule\Infra\RedisLock($redisClient);
$token = $lock->acquire('queue:worker:order_sync', 55); // cron каждую минуту
if ($token === null) {
    return '\\Vendor\\MyModule\\Agent\\OrderSyncAgent::run();';
}

try {
    \Vendor\MyModule\Service\QueueWorker::processBatch(50);
} finally {
    $lock->release('queue:worker:order_sync', $token);
}
```

Практика:

1. TTL lock должен быть чуть меньше интервала cron.
2. Для долгих батчей добавляйте продление TTL (heartbeat).
3. На каждый тип задач используйте отдельный lock-ключ.

---

## Миграция без даунтайма: хиты -> cron -> очередь (с rollback-планом)

### Этап 1. Подготовка

1. Добавьте таблицу очереди и воркер, не выключая старый путь.
2. Оберните новый путь feature-flag.
3. Подключите метрики: queue depth, oldest age, failed rate.

### Этап 2. Перенос запуска на cron

1. Включите запуск через `cron_events.php` (1–5 минут).
2. Отключите зависимость целевых агентов от хитов.
3. Включите защиту от дублей: SQL `claimBatch` или Redis-lock.

### Этап 3. Поэтапное включение очереди

1. Переведите 10% потока на новую схему.
2. Проверьте SLA, failed rate и влияние на P95 веба.
3. Увеличивайте долю: 10% -> 50% -> 100%.

### Этап 4. Стабилизационное окно

1. Оставьте fallback-флаг минимум на 1–2 недели.
2. Мониторьте хвост и долю failed в пиковые окна.
3. После стабильности удаляйте legacy-путь.

### Rollback-план

Если после переключения растут хвост и ошибки:

1. Мгновенно возвращайте feature-flag на старый путь.
2. Снижайте батч/частоту запуска нового воркера.
3. Разбирайте failed/retry и локализуйте узкое место (БД/API/код).
4. Повторяйте rollout с меньшей долей трафика.

Главное правило: rollback должен выполняться одной операцией, без ручного редактирования кода в инциденте.

---

## `addBackgroundJob`: когда уместно, а когда опасно

Подходит:

1. Отправить письмо после регистрации.
2. Обновить вторичный лог/метрику.
3. Запустить короткую пост-обработку файла.

Не подходит:

1. Длинный импорт.
2. Массивные интеграции.
3. Любая задача, где нельзя потерять выполнение.

Важно: `addBackgroundJob` работает в рамках текущего процесса и не дает гарантий как очередь с хранилищем.

Пример:

```php
<?php
$app = \Bitrix\Main\Application::getInstance();
$app->addBackgroundJob(
    [\Vendor\MyModule\Service\EmailService::class, 'sendWelcome'],
    ['user@example.com', 'Добро пожаловать!'],
    \Bitrix\Main\Application::JOB_PRIORITY_NORMAL
);
```

---

## SLA/SLO для фоновых задач: что согласовать с бизнесом до релиза

Чтобы не спорить «быстро/медленно», фиксируйте численные цели:

1. SLO latency: 95% задач типа `order_sync` обрабатываются < 3 минут.
2. SLO reliability: доля `failed` после всех retry < 0.5%.
3. Recovery objective: после деградации очередь возвращается в норму за X минут.

Только после этого можно выбирать батч, интервал cron и число воркеров.

---

## Наблюдаемость фоновой обработки: обязательные метрики продакшена

Управлять фоном без метрик невозможно. В ежедневном мониторинге должны быть минимум пять сигналов: размер очереди (`new + retry + processing`), возраст самой старой задачи, доля `failed`, длительность обработки батча и фактическая частота запусков агента.  
Если этих данных нет, инцидент сначала придет в продукт, а не в мониторинг.

---

## Реальный шаблон алерт-порогов (стартовая конфигурация)

Для большинства e-commerce проектов можно стартовать так:

1. `queue_depth > 5000` дольше 10 минут -> warning.
2. `oldest_task_age > 15 минут` -> critical.
3. `failed_rate > 2%` за 15 минут -> warning.
4. `agent_last_run > 3 * cron_interval` -> critical.

Дальше пороги калибруются по фактическому трафику и SLA.

---

## Алертинг: минимальный контур раннего обнаружения

Пороговые алерты должны фиксировать не факт «что-то случилось», а риск срыва SLA. Практический минимум: переполнение очереди дольше допустимого окна, превышение возраста старейшей задачи, всплеск `failed` и отсутствие запусков агента в ожидаемом интервале.  
Даже базовый контур в Telegram сильно снижает риск ночных инцидентов.

---

## Incident response: что делать при остановке фонового контура

В инциденте важно быстро стабилизировать систему, а не «лечить наугад». Рабочая последовательность такая: ограничить приток новых тяжелых задач, локализовать узкое место (БД, внешний API, код воркера), временно снизить давление на систему размером батча и частотой запусков, затем отделить критичный поток от второстепенного. После стабилизации обязателен RCA с корректирующими мерами.  
Худшее решение в этот момент — просто увеличить таймауты и надеяться, что «само пройдет».

---

## Архитектурные антипаттерны, которые приводят к сбоям

Есть решения, которые на старте выглядят «быстро и удобно», но в продакшене почти гарантируют деградацию: логика агента в `init.php`, один агент на весь объем задач, отсутствие retry/backoff и `failed`-контура, запуск критичных операций на хитах без `cron`, работа без метрик и алертов.  
На нагруженном проекте любой из этих антипаттернов рано или поздно превращается в инцидент.

---

## Release gate: критерии готовности фоновых задач к продакшену

Перед релизом достаточно проверить пять критериев: соблюдение SLA по времени обработки, корректный путь ошибок в `retry/failed`, отсутствие дублей в критичных бизнес-операциях, контролируемая нагрузка от батчей и полноценная наблюдаемость (метрики, логи, алерты).  
Если хотя бы один критерий не проходит, релиз фоновой логики нужно останавливать.

---

## Итог

В Bitrix фоновые задачи работают стабильно только при инфраструктурном подходе: `cron` вместо хитов, агент как оркестратор, очередь для тяжелой обработки, идемпотентность и контур `retry/backoff/failed`, плюс обязательная наблюдаемость.  
Когда эти правила внедрены, команда перестает жить в режиме постоянного тушения и получает предсказуемую эксплуатацию.

---

## Ключевые слова

`1c-bitrix`, `bitrix`, `агенты bitrix`, `cron bitrix`, `background jobs bitrix`, `очереди bitrix`, `caagent`, `retry`, `backoff`, `идемпотентность`, `monitoring`, `incident response`

---

## Автор и контакты

- Telegram: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
