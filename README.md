# Pipes

Lightweight, repeatable AI-assisted development workflow system for DevOps/Kubernetes/IaC engineering tasks.

## Что это

Структурированная система для работы с AI в инженерных задачах. Вместо ad-hoc промптов — чёткий workflow с правилами, чеклистами, playbook'ами и CI-проверками.

## Структура

```
pipes/
├── docs/
│   ├── ai-workflow.md          # 6-stage AI workflow
│   ├── engineering-rules.md    # Engineering rules & guardrails
│   └── pr-checklist.md         # PR review checklist
├── playbooks/
│   ├── new-terraform-module.md # Playbook: new Terraform module
│   ├── new-k8s-controller.md   # Playbook: K8s controller scaffold
│   └── investigate-ci-failure.md # Playbook: CI failure investigation
└── .github/
    └── workflows/
        └── lint.yml            # CI linting pipeline
```

## Принципы

- **Повторяемость** — каждый workflow идёт по одному маршруту
- **Детерминизм** — CI проверяет то, что можно проверить автоматически
- **Человек решает** — AI помогает с рутиной, но архитектурные решения остаются за человеком
- **Простота** — без сложных memory/orchestration слоёв

## Quick Start

```bash
# Install pre-commit hooks
pip install pre-commit
pre-commit install
```

## License

MIT
