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

## Зависимости

### Скиллы из этого репозитория

Для полного workflow устанавливаются все четыре скилла:

- `tatar-orchestrate-task` вызывает `implement`, `tatar-review-ticket` и `tatar-thermonuclear-review`;
- `tatar-review-ticket` использует собственные role references из своего каталога;
- `tatar-thermonuclear-review` выполняет независимый финальный аудит;
- `implement` выполняет изменения и коммитит их в назначенную ветку.

Устанавливать только `tatar-orchestrate-task` без остальных трёх нельзя: перед изменением Linear он проверяет наличие обязательных reviewer- и implementer-скиллов.

### Внешние скиллы

Standalone-скилл `implement` ссылается на:

- `tdd` — желательно для реализации test-first на заранее выбранных seams;
- `code-review` — финальное ревью самостоятельной реализации.

При запуске через `tatar-orchestrate-task`:

- `tdd` используется имплементором, когда применимо и доступно;
- финальный `code-review` намеренно не запускается;
- централизованное ревью выполняют `tatar-review-ticket` и `tatar-thermonuclear-review`.

Поэтому для основного orchestration workflow `tdd` полезен, но не является жёсткой зависимостью, а внешний `code-review` не требуется.

### Интеграции и runtime

Основной оркестратор требует:

- Linear API/connector с read/write-доступом к parent issue и descendants;
- статусы команды Linear с точными именами `In Progress` и `In Review`;
- runtime с поддержкой независимых subagents и параллельных reviewer agents;
- Git с поддержкой branches, commits, cherry-pick/merge и linked worktrees;
- доступ на запись к локальной файловой системе для создания временных worktrees;
- проектные `AGENTS.md` и `_bmad-output/project-context.md`;
- тестовые, lint, typecheck и migration-команды целевого проекта;
- Graphify CLI и `graphify-out/`, если политика целевого репозитория требует финальную регенерацию графа.

Оркестратор не поставляет Linear connector, subagent runtime, Git, Graphify или проектные toolchains — они должны быть доступны в среде Codex и целевом репозитории.

### GitHub-доступ для установки

Репозиторий private. Для установки нужен один из вариантов:

- активная аутентификация `gh auth login`;
- Git credentials с доступом к `tatarrr95/tatar-orchestrate-task`;
- `GITHUB_TOKEN` или `GH_TOKEN` с правом чтения репозитория.

## Запуск

```text
/tatar-orchestrate-task ALT-123
```
