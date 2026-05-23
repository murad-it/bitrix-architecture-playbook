# Bitrix Architecture Playbook

Практический плейбук по архитектуре и производительности в 1C-Битрикс: DDD, Vertical Slice, SOLID и инженерные подходы к оптимизации.

## Для кого этот репозиторий

- Backend-разработчики на PHP/1C-Битрикс (middle/senior)
- Техлиды и архитекторы
- Команды, которым нужен предсказуемый рост качества кода и производительности

## Что внутри

### 1) Архитектурные принципы в Bitrix
- [SOLID, KISS и YAGNI в 1C-Битрикс: практическое руководство](./bitrix-solid-kiss-yagni-practical-guide-ru.md)
- [Vertical Slice и DDD в 1C-Битрикс: практическая архитектура](./bitrix-vertical-slice-ddd-practical-guide-ru.md)

### 2) Производительность и диагностика
- [Производительность 1C-Битрикс на практике: 10 системных ошибок и как их устранять](./bitrix-performance-optimization-10-mistakes-practical-guide-ru.md)
- [D7 ORM в 1C-Битрикс на production: как писать быстрый код, а не красивую деградацию](./bitrix-d7-orm-performance-runtime-relations-practical-guide-ru.md)
- [Кеширование в 1C-Битрикс на production: полная практическая система](./bitrix-caching-complete-practical-guide-ru.md)

## Подход в статьях

- Минимум теории без пользы, максимум применимости в production.
- Примеры ориентированы на профессиональный D7-подход (ORM, сервисный слой, корректное кэширование).
- Каждый разбор строится по схеме: симптом -> причина -> решение -> проверка результата.

## Почему это полезно

- Быстрее проводить code review и архитектурные обсуждения.
- Снижать количество регрессий при доработках.
- Ускорять проект за счёт системных, а не случайных оптимизаций.

## Рекомендуемый порядок чтения

1. Начать с SOLID/KISS/YAGNI для выравнивания инженерных принципов.
2. Перейти к Vertical Slice + DDD для структуры фич и границ слоёв.
3. Закрепить на статье по производительности с метриками и диагностиками.
4. Углубиться в D7 ORM статью по runtime relation, JOIN и индексам.
5. Закрыть системную тему кеширования на уровне production-архитектуры.

## Ключевые темы

`1c-bitrix`, `bitrix`, `php`, `backend`, `software-architecture`, `ddd`, `domain-driven-design`, `vertical-slice`, `solid-principles`, `clean-code`, `performance-optimization`, `d7-orm`, `mysql-indexes`, `runtime-relations`, `bitrix-cache`, `managed-cache`, `tagged-cache`

## Где меня найти

- Telegram-канал: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)

---

## English

Practical playbook on 1C-Bitrix architecture and performance: DDD, Vertical Slice, SOLID, and production-grade optimization patterns.

### Who This Repository Is For

- PHP/1C-Bitrix backend developers (middle/senior)
- Tech leads and solution architects
- Teams that need predictable code quality and performance growth

### Contents

#### 1) Architecture Principles in Bitrix
- [SOLID, KISS and YAGNI in 1C-Bitrix: Practical Guide (RU)](./bitrix-solid-kiss-yagni-practical-guide-ru.md)
- [Vertical Slice and DDD in 1C-Bitrix: Practical Architecture (RU)](./bitrix-vertical-slice-ddd-practical-guide-ru.md)

#### 2) Performance and Diagnostics
- [1C-Bitrix Performance in Practice: 10 Systemic Mistakes and How to Fix Them (RU)](./bitrix-performance-optimization-10-mistakes-practical-guide-ru.md)
- [D7 ORM in 1C-Bitrix Production: Fast Queries, Runtime Relations, JOIN Strategy, and MySQL Indexing (RU)](./bitrix-d7-orm-performance-runtime-relations-practical-guide-ru.md)
- [1C-Bitrix Caching in Production: Complete Practical System (RU)](./bitrix-caching-complete-practical-guide-ru.md)

### Article Approach

- Production value over abstract theory.
- Professional D7-oriented examples (ORM, service layer, robust caching).
- Clear flow in every chapter: symptom -> root cause -> solution -> verification.

### Topics

`1c-bitrix`, `bitrix`, `php`, `backend`, `software-architecture`, `ddd`, `domain-driven-design`, `vertical-slice`, `clean-code`, `performance-optimization`, `d7-orm`, `mysql-indexes`, `runtime-relations`, `bitrix-cache`, `managed-cache`, `tagged-cache`

### Connect

- Telegram: [@murad_pro_it](https://t.me/murad_pro_it)
- Instagram: [@murad__it](https://www.instagram.com/murad__it/)
