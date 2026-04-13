# Playbook: New Terraform Module

Пошаговый процесс создания нового переиспользуемого Terraform модуля.
Используй этот playbook каждый раз, когда создаёшь модуль с нуля.

---

## Context

**Когда использовать:** Нужен новый Terraform модуль для provisioning ресурсов (AWS, GCP, Azure).
**Prerequisite:** Terraform >= 1.5, доступ к remote state backend, понимание целевого провайдера.

---

## Steps

### 1. Define scope

- Определить, что модуль создаёт (один логический ресурс или группу связанных ресурсов)
- Записать Goal Statement (см. `docs/ai-workflow.md`, Stage 1)
- Определить, какие параметры будут настраиваемыми (variables), а какие — фиксированными

### 2. Create directory structure

```
modules/{module-name}/
├── main.tf          # Resource definitions
├── variables.tf     # Input variables
├── outputs.tf       # Module outputs
├── versions.tf      # Provider & Terraform version constraints
├── locals.tf        # Computed values (optional)
├── data.tf          # Data sources (optional)
├── README.md        # Documentation
└── tests/           # Tests directory
    └── main.tftest.hcl  # Native Terraform test (или terratest)
```

### 3. Implement `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Зафиксировать major version провайдера. Не использовать `>=` без upper bound.

### 4. Define `variables.tf`

- Каждая переменная: `description`, `type`, `validation` (где уместно)
- Обязательные переменные — без `default`
- Sensitive переменные помечены `sensitive = true`
- Naming: `snake_case`, descriptive (Rule 4, 5)

```hcl
variable "environment" {
  description = "Deployment environment (staging, production)"
  type        = string

  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Environment must be 'staging' or 'production'."
  }
}
```

### 5. Implement `main.tf`

- Один resource block — один ресурс
- Использовать `locals` для computed значений вместо inline expressions
- Добавить `precondition`/`postcondition` для критичных ресурсов (Rule 10)
- Tags на каждый ресурс, включающие `environment`, `managed_by = "terraform"`, `module`

### 6. Define `outputs.tf`

- Output для каждого значения, которое нужно другим модулям/environments
- Всегда включать `description`
- Sensitive outputs помечены `sensitive = true`

### 7. Write README.md

Минимум:
- Что делает модуль
- Usage example с реальными значениями
- Inputs table (или ссылка на `terraform-docs` output)
- Outputs table
- Prerequisites / dependencies

### 8. Validate

```bash
# Форматирование
terraform fmt -check -recursive

# Валидация
terraform init -backend=false
terraform validate

# Linting
tflint --init
tflint

# Security scan
tfsec . # или checkov -d .
```

### 9. Test

```bash
# Native Terraform test (>= 1.6)
terraform test

# Или terratest (Go)
go test -v -timeout 30m ./tests/
```

### 10. Integrate

- Добавить вызов модуля в `environments/{env}/main.tf`
- Прогнать `terraform plan` — убедиться, что plan выглядит ожидаемо
- Пройти PR checklist (`docs/pr-checklist.md`)

---

## Verification

- [ ] `terraform fmt -check` — чисто
- [ ] `terraform validate` — success
- [ ] `tflint` — no errors
- [ ] README содержит usage example
- [ ] Все variables имеют `description`
- [ ] Все outputs имеют `description`
- [ ] Security scan пройден (no HIGH/CRITICAL findings)

---

## Common Pitfalls

- **Слишком широкий scope модуля** — если модуль делает больше одной "вещи", разбей на два
- **Hardcoded provider config** — provider configuration должна быть в calling module, не в самом модуле
- **Отсутствие version constraints** — всегда фиксируй major version в `versions.tf`
- **`count` vs `for_each`** — предпочитай `for_each` (stable keys vs fragile indices)
- **Забыть про state** — убедись, что state backend настроен до первого apply
