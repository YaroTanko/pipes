# PR Checklist

Repeatable gate перед merge. Каждый PR проходит через этот чеклист — и автор, и ревьюер сверяются с ним. Не импровизация, а процедура.

---

## Requirements

- [ ] PR решает заявленную задачу (issue/ticket linkage)
- [ ] Scope не вышел за пределы исходного Goal Statement
- [ ] Все acceptance criteria из задачи выполнены
- [ ] Если scope изменился — это задокументировано и согласовано

---

## Code Quality

- [ ] Код следует engineering rules (`docs/engineering-rules.md`)
- [ ] Нет TODO-заглушек, оставленных "на потом"
- [ ] Naming conventions соблюдены (Rule 4, 5)
- [ ] Commit messages в формате Conventional Commits (Rule 6)
- [ ] Нет дублирования — переиспользуемые части вынесены в модули/функции

---

## Backward Compatibility

- [ ] Существующие API/interfaces не сломаны
- [ ] Terraform: `terraform plan` не показывает unexpected destroy/recreate
- [ ] Helm: values по умолчанию не изменились без явного обоснования
- [ ] K8s: не удалены labels/annotations, от которых зависят другие системы
- [ ] Миграция данных предусмотрена (если applicable)

---

## Security

- [ ] Нет hardcoded secrets, токенов, паролей (Rule 12)
- [ ] IAM/RBAC — минимальные permissions (Rule 13)
- [ ] Pod Security Standards соблюдены (Rule 14)
- [ ] Network policies не ослаблены без обоснования
- [ ] Sensitive данные не попадают в логи
- [ ] Docker images используют конкретные теги, не `latest`

---

## Observability

- [ ] Health endpoints добавлены/обновлены (Rule 8)
- [ ] K8s labels на месте (Rule 9)
- [ ] Логирование добавлено для новых code paths (Rule 7)
- [ ] Alerts/dashboards обновлены (если поведение изменилось)
- [ ] Terraform outputs включают endpoints/ARNs для мониторинга

---

## Tests

- [ ] Новый код покрыт тестами (Rule 15)
- [ ] Существующие тесты не сломаны
- [ ] Lint пройден: `terraform fmt`, `helm lint`, `yamllint`, `shellcheck`
- [ ] Validate пройден: `terraform validate`, `kubeconform`
- [ ] CI pipeline зелёный (Rule 16)

---

## Documentation

- [ ] README обновлён (Rule 19, 20)
- [ ] Input/output variables задокументированы
- [ ] Breaking changes описаны в PR description
- [ ] Usage examples актуальны

---

## Final Checks

- [ ] Self-review проведён (diff прочитан целиком)
- [ ] PR description содержит контекст: что, почему, как
- [ ] Размер PR разумный (< 400 строк, иначе разбить)
- [ ] Branch up to date с main
- [ ] Все conversations resolved

---

## How to use

**Автор PR:**
1. Пройти чеклист перед запросом review
2. Отметить все пункты в PR description (копировать чеклист в PR template)
3. Если пункт не применим — отметить с пометкой "N/A" и причиной

**Ревьюер:**
1. Проверить, что автор прошёл чеклист
2. Сверить каждый пункт со своей стороны
3. Не approve, пока все применимые пункты не отмечены
