# Playbook: Investigate CI Failure

Алгоритм диагностики CI failures. Вместо случайного "пофиксить и перезапустить" — системный подход к поиску причины.

---

## Context

**Когда использовать:** CI pipeline упал, и причина не очевидна из первого взгляда на лог.
**Prerequisite:** Доступ к CI системе (GitHub Actions, GitLab CI), понимание pipeline structure.

---

## Steps

### 1. Identify the failed step

- Открой CI лог и найди **первый** упавший step
- Не смотри на последний step — он часто падает как следствие предыдущей ошибки
- Запиши: название step'а, exit code, первые строки ошибки

### 2. Classify the failure

Определи тип ошибки:

**a) Compilation / syntax error**
- Ошибка в коде: синтаксис, типы, import
- Действие: смотри diff последнего коммита, ищи что сломалось

**b) Test failure**
- Тест упал с assertion error
- Действие: прочитай assertion message, найди расхождение expected vs actual

**c) Lint / format error**
- Код не соответствует стилю
- Действие: запусти linter локально, поправь

**d) Dependency error**
- Не удалось скачать зависимость, version conflict
- Действие: проверь lockfile, registry availability, network

**e) Infrastructure / environment error**
- Таймаут, out of memory, unavailable service
- Действие: это не баг в коде — проверь CI runner, quotas, external dependencies

**f) Flaky test**
- Тест иногда проходит, иногда нет
- Действие: запусти 3 раза. Если нестабильно — это flaky, чини тест, не skip'ай

### 3. Reproduce locally

```bash
# Для каждого типа:

# Terraform
terraform init && terraform validate && terraform plan

# Helm
helm lint ./charts/my-chart && helm template ./charts/my-chart | kubeconform

# Go
go build ./... && go test ./... -v

# Generic
# Скопируй команду из CI лога и запусти локально с теми же аргументами
```

Если локально проходит, а в CI нет — разница в окружении. Проверь:
- Версии tools (CI может использовать другую версию)
- Environment variables
- Файловая система (case sensitivity на Linux vs macOS)
- Network access (CI runner может не иметь доступа к внутренним ресурсам)

### 4. Find root cause

```
Ошибка → Какой файл → Какая строка → Что изменилось последним → Почему
```

Инструменты:
- `git log --oneline -5` — что менялось недавно
- `git bisect` — если непонятно, какой коммит сломал
- `git diff main..HEAD` — полный diff текущей ветки

### 5. Fix

- Минимальное исправление, которое решает проблему
- Не фиксить "заодно" другие вещи — это отдельный PR
- Если причина — flaky test, не `skip`, а `fix` (Rule 16)

### 6. Verify fix

```bash
# Запусти тот же step, что упал
# Убедись, что fix не сломал что-то другое
# Прогони полный pipeline локально (если возможно)
```

### 7. Prevent recurrence

Задай вопрос: "Можно ли сделать так, чтобы эта ошибка больше не возникала?"

Варианты:
- Добавить pre-commit hook, который ловит эту ошибку раньше
- Добавить CI step, который проверяет конкретное условие
- Улучшить error message, чтобы следующий разработчик потратил меньше времени
- Обновить документацию / engineering rules

---

## Verification

- [ ] Root cause определён и записан
- [ ] Fix минимальный и целевой
- [ ] CI pipeline зелёный после fix
- [ ] Fix протестирован локально перед push
- [ ] Prevention measure добавлена (если применимо)

---

## Common Pitfalls

- **"Перезапущу pipeline, может пройдёт"** — это маскирует проблему, не решает. Сначала пойми, почему упал.
- **Фиксить симптом, а не причину** — если тест падает из-за race condition, не добавляй `sleep`, фикси race
- **Коммитить "fix ci" без понимания** — слепой trial-and-error создаёт шум в git history
- **Не проверять локально** — если ты не можешь воспроизвести проблему, ты не можешь подтвердить fix
- **Skip flaky test** — это технический долг, который растёт. Лучше потратить 30 минут сейчас.

---

## Decision Tree

```
CI Failed
├── Can I find the first failed step?
│   ├── YES → Read error message → Classify (a-f)
│   └── NO → Check pipeline config (syntax error in CI file?)
│
├── Can I reproduce locally?
│   ├── YES → Debug locally → Fix → Verify
│   └── NO → Environment difference → Check versions, vars, network
│
├── Is it flaky?
│   ├── YES → Run 3x → If unstable, fix the test
│   └── NO → Normal failure → Find root cause
│
└── Fixed?
    ├── YES → Add prevention measure → PR
    └── NO → Escalate / Ask for help (pair debugging)
```
