# Руководство для контрибьютеров

Спасибо за интерес к проекту PlantUML DSL! Здесь вы найдёте информацию о том, как внести свой вклад.

## Как внести свой вклад

### 1. Fork репозитория

Создайте форк проекта через GitHub UI.

### 2. Создайте branch от `main`

```bash
git checkout -b feat/my-new-feature
# или
git checkout -b fix/my-fix
```

Названия веток должны отражать тип изменений:
- `feat/` — новый функционал
- `fix/` — исправление ошибок
- `docs/` — документация
- `refactor/` — рефакторинг

### 3. Внесите изменения

- Следуйте стилю кода проекта
- Добавляйте тесты для нового функционала в `tests/`
- Обновляйте документацию если нужно

### 4. Проверьте изменения локально

```bash
task test     # запустить тесты
task build    # проверить сборку
```

Проект автоматически запускает тесты перед каждым коммитом через git hooks.

### 5. Создайте Pull Request

Отправьте изменения в `main` ветку через GitHub UI. Заполните шаблон PR полностью.

## Commit Messages

Проект использует **Conventional Commits**. Формат:

```
<type>(<scope>): <description>
```

**Типы:**
- `feat` — новый функционал
- `fix` — исправление
- `docs` — документация
- `refactor` — рефакторинг
- `test` — тесты
- `chore` — служебные изменения

**Примеры:**
```
feat: add macro for sequence diagrams
fix: correct parameter name in state diagram
docs: update README with new usage examples
```

Подробнее: https://www.conventionalcommits.org/

## Структура проекта

```
src/        — DSL библиотека (*.puml файлы)
tests/      — Юнит-тесты (*.Tests.puml)
playground/ — Примеры использования
doc/        — Документация
```

## Вопросы?

Если у вас есть вопросы по коду или процессу контрибьюции — создайте issue в репозитории.

## Code of Conduct

Мы следуем [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) для создания дружелюбного и уважительного сообщества.
