# Tatar Orchestrate Task

Набор Codex-скиллов для автономной реализации родительской Linear-задачи с подзадачами, независимыми ролями ревью и финальным структурным аудитом.

## Состав

- `tatar-orchestrate-task` — основной оркестратор задачи и её подзадач.
- `implement` — имплементор работы по спецификации или тикетам.
- `tatar-review-ticket` — изолированные роли `requirements`, `correctness`, `edge-cases` и `integration`.
- `tatar-thermonuclear-review` — финальный аудит архитектурной формы всего интегрированного изменения.

## Установка

Установить все четыре скилла:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo tatarrr95/tatar-orchestrate-task \
  --path \
    skills/tatar-orchestrate-task \
    skills/implement \
    skills/tatar-review-ticket \
    skills/tatar-thermonuclear-review
```

Репозиторий private, поэтому установка использует действующую GitHub-аутентификацию или `GITHUB_TOKEN`/`GH_TOKEN`.

После установки скиллы доступны в следующем сообщении Codex.

## Требования окружения

Основной оркестратор ожидает:

- доступ к Linear;
- поддержку запуска независимых агентов;
- Git с поддержкой worktree;
- проектные `AGENTS.md` и `_bmad-output/project-context.md`;
- статусы Linear с точными именами `In Progress` и `In Review`.

При самостоятельном использовании `implement` также ссылается на внешние `/tdd` и `/code-review`. Внутри `tatar-orchestrate-task` финальный `/code-review` имплементора намеренно заменяется централизованными reviewer-скиллами из этого репозитория.

## Запуск

```text
/tatar-orchestrate-task ALT-123
```
