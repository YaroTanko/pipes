# Engineering Rules

Набор правил, которые AI и инженер обязаны соблюдать при реализации задач.
Эти правила — "память проекта": вместо того чтобы каждый раз объяснять AI одни и те же ограничения, они зафиксированы один раз и применяются автоматически.

---

## Repository Structure

### Rule 1: Consistent directory layout

Каждый проект должен следовать стандартной структуре:
- `docs/` — документация
- `modules/` или `charts/` — переиспользуемые компоненты
- `environments/` или `clusters/` — конфигурация по окружениям
- `tests/` — тесты
- `scripts/` — вспомогательные скрипты (всегда с `set -euo pipefail`)

### Rule 2: One module/chart = one directory

Каждый Terraform модуль, Helm chart или K8s manifest set живёт в отдельной директории с собственным README.

### Rule 3: No generated files in version control

Не коммитить: `.terraform/`, `node_modules/`, `__pycache__/`, `*.tfstate`, lockfiles для IaC (кроме `package-lock.json` и `go.sum`).

---

## Naming Conventions

### Rule 4: Snake_case for IaC, kebab-case for K8s

- Terraform resources, variables, outputs → `snake_case`
- K8s resource names, labels, annotations → `kebab-case`
- Helm values → `camelCase` (стандарт Helm)

### Rule 5: Descriptive, contextual names

Имена ресурсов должны включать контекст: `{project}-{component}-{env}`.
Не допускать: `main`, `default`, `test`, `temp` как имена ресурсов.

### Rule 6: Commit messages follow Conventional Commits

Формат: `type(scope): description`
Типы: `feat`, `fix`, `docs`, `ci`, `refactor`, `test`, `chore`.
Scope = модуль или компонент, которого касается изменение.

---

## Logging & Observability

### Rule 7: Structured logging only

Все приложения пишут логи в JSON-формате. Обязательные поля: `timestamp`, `level`, `message`, `service`.
Для shell-скриптов: каждый `echo` сопровождается контекстом (что делает скрипт, на каком шаге).

### Rule 8: Every service exposes health and metrics

- `/healthz` или `/readyz` для liveness/readiness probes
- `/metrics` в формате Prometheus (если приложение)
- Для IaC: output'ы, которые позволяют подключить мониторинг (endpoint, ARN, etc.)

### Rule 9: Labels and annotations on every K8s resource

Обязательные labels:
- `app.kubernetes.io/name`
- `app.kubernetes.io/version`
- `app.kubernetes.io/managed-by`
- `app.kubernetes.io/part-of`

---

## Error Handling

### Rule 10: Fail fast, fail loud

- Shell-скрипты: `set -euo pipefail` в начале каждого файла
- Terraform: `precondition` и `postcondition` blocks для критичных ресурсов
- Не глотать ошибки. Если операция может упасть — обработать явно или дать упасть.

### Rule 11: No silent defaults

Каждая переменная без default должна быть обязательной (`nullable = false` в Terraform).
Не использовать "магические" default-значения, которые могут сломать production.

---

## Security

### Rule 12: No secrets in code, ever

Secrets — только через:
- Environment variables (в runtime)
- External secret managers (AWS Secrets Manager, Vault, External Secrets Operator)
- SOPS для encrypted files в git

Hardcoded пароли, токены, API keys — блокер для merge.

### Rule 13: Least privilege by default

- IAM roles/policies: минимальные permissions, никаких `*` в actions или resources
- K8s RBAC: роли на уровне namespace, не cluster-wide без явной необходимости
- Network policies: default deny, explicit allow

### Rule 14: Pod Security Standards

Все workloads соответствуют `restricted` Pod Security Standard:
- `runAsNonRoot: true`
- `readOnlyRootFilesystem: true`
- `allowPrivilegeEscalation: false`
- Без `hostPath`, `hostNetwork`, `hostPID`

---

## Testing

### Rule 15: Every module has at least one test

- Terraform: `terraform validate` + `terraform plan` как минимум, `terratest` для critical modules
- Helm: `helm lint` + `helm template` + `helm test`
- K8s manifests: `kubeconform` или `conftest` с OPA policies
- Shell scripts: `shellcheck`

### Rule 16: Tests run in CI, not just locally

Каждый PR автоматически проходит lint + validate + test pipeline.
Нет исключений из CI. Если тест flaky — починить, а не скипнуть.

---

## IaC Guardrails

### Rule 17: Validate before apply

Обязательная цепочка для Terraform:
```
terraform fmt -check → terraform validate → tflint → tfsec/checkov → terraform plan
```

Для Helm:
```
helm lint → helm template | kubeconform → conftest (если есть policies)
```

### Rule 18: State management rules

- Remote state only (S3, GCS, Terraform Cloud) — никогда local state для shared ресурсов
- State locking включён (DynamoDB для S3 backend)
- Один state file на один environment + один logical component
- Никогда не редактировать state вручную без крайней необходимости

---

## Documentation

### Rule 19: README in every module directory

Минимальный README:
- Что делает модуль/chart
- Inputs/outputs (или ссылка на сгенерированные docs)
- Usage example
- Prerequisites

### Rule 20: Update docs with code

Если PR меняет поведение — docs обновляются в том же PR.
Не допускать "я обновлю docs потом" — это не происходит.

---

## How to use these rules

1. **AI:** Перед началом Stage 5 (Implement) — прочитать все правила и проверить, что plan им не противоречит
2. **Engineer:** При review — сверяться с правилами как с checklist
3. **CI:** Автоматизировать всё, что можно проверить программно (Rules 3, 6, 12, 15-18)
4. **Эволюция:** Если правило устарело — обновить через PR, а не игнорировать
