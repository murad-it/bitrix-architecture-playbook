# 1C-Битрикс: Архитектура, Производительность и Надёжность

Инженерный playbook для production-проектов: системная архитектура, быстрый код и предсказуемая эксплуатация.

Этот репозиторий собран как инженерный playbook: без теоретической воды, с фокусом на решения, которые выдерживают нагрузку, релизы и реальную эксплуатацию.

## Кому полезно

- Разработчикам, которые устали «чинить по факту» и хотят проектировать Bitrix-решения сразу правильно
- Техлидам и архитекторам, которым нужен единый инженерный стандарт для команды
- Командам, которые хотят ускорять релизы без роста регрессий и ночных инцидентов

## Что внутри

### Архитектура и инженерные принципы

- [SOLID, KISS и YAGNI в 1C-Битрикс: практическое руководство](./bitrix-solid-kiss-yagni-practical-guide-ru.md)
- [Vertical Slice и DDD в 1C-Битрикс: практическая архитектура](./bitrix-vertical-slice-ddd-practical-guide-ru.md)

### Производительность, данные и эксплуатация

- [Производительность 1C-Битрикс на практике: 10 системных ошибок и как их устранять](./bitrix-performance-optimization-10-mistakes-practical-guide-ru.md)
- [D7 ORM в 1C-Битрикс на production: как писать быстрый код, а не красивую деградацию](./bitrix-d7-orm-performance-runtime-relations-practical-guide-ru.md)
- [Кеширование в 1C-Битрикс на production: полная практическая система](./bitrix-caching-complete-practical-guide-ru.md)
- [Агенты, cron и очереди в 1C-Битрикс: production-архитектура надежной фоновой обработки](./bitrix-agents-background-jobs-cron-queue-practical-guide-ru.md)

## Принципы материалов

- Каждая статья ориентирована на production-решения, а не на учебные демо.
- Разборы строятся вокруг цепочки: проблема -> причина -> решение -> проверка.
- В коде и примерах учитываются эксплуатационные риски: нагрузка, инвалидация, SLA, отказоустойчивость.

## Как использовать в команде

1. Использовать статьи как базовый стандарт для code review и архитектурных решений.
2. Фиксировать принятые паттерны во внутренних стандартах команды.
3. Перед релизами проходить по релевантным разделам как по preflight-проверке.

## Рекомендуемый порядок чтения

1. SOLID/KISS/YAGNI
2. Vertical Slice + DDD
3. Производительность (10 ошибок)
4. D7 ORM
5. Кеширование
6. Агенты, cron и очереди

## Что это даст проекту

1. Меньше спорных решений «по ощущениям» и больше инженерной предсказуемости.
2. Быстрее онбординг разработчиков в Bitrix-архитектуру команды.
3. Ниже риск регрессий в производительности и фоновой обработке после релизов.

## Темы репозитория

`1c-bitrix`, `bitrix`, `php`, `backend`, `software-architecture`, `ddd`, `vertical-slice`, `solid-principles`, `clean-code`, `performance-optimization`, `d7-orm`, `mysql`, `bitrix-cache`, `managed-cache`, `tagged-cache`, `bitrix-agents`, `bitrix-cron`, `bitrix-queue`, `reliability`

## Контакты

- Telegram: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)

---

## English (Short Version)

Production-focused knowledge base for 1C-Bitrix architecture, performance, and reliability.

### Audience

- PHP/1C-Bitrix backend engineers (middle/senior)
- Tech leads and solution architects
- Teams that need predictable quality, performance, and release stability

### Contents

#### Architecture and Engineering Fundamentals

- [SOLID, KISS and YAGNI in 1C-Bitrix: Practical Guide (RU)](./bitrix-solid-kiss-yagni-practical-guide-ru.md)
- [Vertical Slice and DDD in 1C-Bitrix: Practical Architecture (RU)](./bitrix-vertical-slice-ddd-practical-guide-ru.md)

#### Performance, Data, and Operations

- [1C-Bitrix Performance in Practice: 10 Systemic Mistakes and How to Fix Them (RU)](./bitrix-performance-optimization-10-mistakes-practical-guide-ru.md)
- [D7 ORM in 1C-Bitrix Production: Fast Queries, Runtime Relations, JOIN Strategy, and MySQL Indexing (RU)](./bitrix-d7-orm-performance-runtime-relations-practical-guide-ru.md)
- [1C-Bitrix Caching in Production: Complete Practical System (RU)](./bitrix-caching-complete-practical-guide-ru.md)
- [1C-Bitrix Agents, Cron and Queues: Production Architecture for Reliable Background Processing (RU)](./bitrix-agents-background-jobs-cron-queue-practical-guide-ru.md)

### Repository Approach

- Production-first patterns over abstract theory
- Problem -> root cause -> solution -> verification structure
- Operational reliability as a core concern: load, invalidation, SLA, failure handling

### Topics

`1c-bitrix`, `bitrix`, `php`, `backend`, `software-architecture`, `ddd`, `vertical-slice`, `performance-optimization`, `d7-orm`, `mysql`, `bitrix-cache`, `bitrix-agents`, `bitrix-cron`, `bitrix-queue`, `reliability`

### Contacts

- Telegram: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
