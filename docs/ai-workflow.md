# AI Workflow

Структурированный процесс работы с AI для инженерных задач (DevOps, Kubernetes, IaC).
Каждая задача проходит 6 стадий последовательно. Переход на следующую стадию — только после выполнения критериев текущей.

## Overview

```
Goal → Constraints → Design → Plan → Implement → Verify
```

Основная идея: AI не прыгает сразу в код, а проходит тот же маршрут, что и инженер. Это снижает количество ошибок и повышает предсказуемость результата.

---

## Stage 1: Goal

**Что делаем:** Фиксируем цель задачи одним-двумя предложениями.

**Входные данные:**
- Описание задачи от пользователя (issue, ticket, устный запрос)
- Контекст проекта (какой репо, какой стек)

**Выходные данные:**
- Чёткая формулировка цели (Goal Statement)
- Список ключевых stakeholders или систем, на которые повлияет изменение

**Критерии перехода:**
- [ ] Цель сформулирована и однозначна
- [ ] Понятно, что НЕ входит в scope задачи

**Пример для DevOps:**
> Goal: Создать Terraform модуль для provisioning Redis cluster в AWS ElastiCache с поддержкой multi-AZ.
> Out of scope: Мониторинг, application-level конфигурация.

---

## Stage 2: Constraints

**Что делаем:** Собираем ограничения — технические, организационные, по безопасности.

**Входные данные:**
- Goal Statement из Stage 1
- Существующая инфраструктура / кодовая база
- Требования безопасности и compliance

**Выходные данные:**
- Список ограничений с категоризацией:
  - **Technical:** версии, совместимость, зависимости
  - **Security:** secrets management, RBAC, network policies
  - **Organizational:** naming conventions, team ownership, approval процессы
  - **Performance:** SLA, resource limits, scaling requirements

**Критерии перехода:**
- [ ] Все известные ограничения задокументированы
- [ ] Нет конфликтующих ограничений (или конфликты разрешены)

**Пример для K8s:**
> - Technical: K8s 1.28+, Helm 3, без CRD от сторонних операторов
> - Security: Pod Security Standards restricted, no hostPath
> - Organizational: Namespace naming `{team}-{env}`, labels по стандарту компании

---

## Stage 3: Design

**Что делаем:** Проектируем решение на высоком уровне, без деталей реализации.

**Входные данные:**
- Goal Statement
- Constraints list
- Существующая архитектура (если есть)

**Выходные данные:**
- Архитектурная схема или описание компонентов
- Список файлов/модулей, которые будут созданы или изменены
- Выбранные технологии / подходы с обоснованием
- Альтернативы, которые рассматривались (и почему отброшены)

**Критерии перехода:**
- [ ] Design не нарушает ни одного constraint
- [ ] Понятно, какие файлы будут затронуты
- [ ] Trade-offs задокументированы

**Пример для IaC:**
> Design: Terraform module в `modules/elasticache-redis/` с variables для node type, num replicas, subnet group.
> Альтернатива: Использовать Crossplane — отброшено, т.к. нет Crossplane в кластере.

---

## Stage 4: Plan

**Что делаем:** Разбиваем design на конкретные шаги реализации.

**Входные данные:**
- Design из Stage 3
- Existing codebase / file structure

**Выходные данные:**
- Пронумерованный список шагов
- Для каждого шага: что делать, какой файл, какой результат
- Порядок выполнения (dependencies между шагами)
- Estimated effort (S/M/L)

**Критерии перехода:**
- [ ] Каждый шаг достаточно конкретный для реализации
- [ ] Нет пропущенных шагов (например, забытые migrations, config changes)
- [ ] Шаги можно выполнять по порядку без блокеров

**Пример:**
> 1. Создать `modules/elasticache-redis/main.tf` — resource definitions [M]
> 2. Создать `modules/elasticache-redis/variables.tf` — input variables [S]
> 3. Создать `modules/elasticache-redis/outputs.tf` — module outputs [S]
> 4. Добавить вызов модуля в `environments/staging/main.tf` [S]
> 5. Написать `modules/elasticache-redis/README.md` [S]
> 6. Добавить terratest в `tests/elasticache_test.go` [M]

---

## Stage 5: Implement

**Что делаем:** Реализуем план шаг за шагом.

**Входные данные:**
- Plan из Stage 4
- Codebase

**Выходные данные:**
- Рабочий код / конфигурация
- Коммит(ы) с осмысленными сообщениями

**Правила реализации:**
- Один шаг плана — один логический коммит
- Не отклоняться от плана без явного обоснования
- Если в процессе реализации обнаружена проблема с design — вернуться на Stage 3
- Следовать engineering rules (см. `engineering-rules.md`)

**Критерии перехода:**
- [ ] Все шаги плана выполнены
- [ ] Код компилируется / проходит `terraform validate` / `helm lint`
- [ ] Нет TODO-заглушек в коде

---

## Stage 6: Verify

**Что делаем:** Проверяем результат перед отправкой на review.

**Входные данные:**
- Готовый код из Stage 5
- PR Checklist (см. `pr-checklist.md`)

**Проверки:**
1. **Lint & format:** `terraform fmt`, `helm lint`, `yamllint`, `markdownlint`
2. **Tests:** unit tests, integration tests (если применимо)
3. **Security:** no hardcoded secrets, proper RBAC, least privilege
4. **Docs:** README обновлён, comments в коде актуальны
5. **CI:** pipeline зелёный
6. **PR Checklist:** все пункты отмечены

**Критерии завершения:**
- [ ] Все автоматические проверки пройдены
- [ ] PR checklist полностью заполнен
- [ ] Self-review проведён (diff прочитан целиком)

---

## Когда возвращаться назад

- **Implement → Design:** Обнаружена архитектурная проблема, которую план не учитывает
- **Verify → Implement:** Найден баг или lint error, который требует исправления кода
- **Verify → Constraints:** В процессе verify обнаружено нарушение ограничений
- **Any → Goal:** Scope creep — задача выросла за пределы исходной цели

## Anti-patterns

- ❌ Прыгать сразу в Stage 5 (Implement) без прохождения Stage 1-4
- ❌ Пропускать Stage 6 (Verify), надеясь, что "CI поймает"
- ❌ Автоматизировать архитектурные решения (Stage 3) — это всегда решение человека
- ❌ Одновременно менять goal и implementation
