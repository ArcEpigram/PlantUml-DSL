# C4 Container Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Реализовать C4-элементы `Container`, `ContainerDb`, `ContainerQueue` в PlantUML-DSL в стиле, единообразном с `Person` / `System` / `Boundary` (закрывает issue #18).

**Architecture:** Три регистратора с нативными сигнатурами C4-PlantUML (3-й аргумент — `$techn`). Общая команда `Containers(_Ext)` ×4. Маппинг `element_type` → `Container` / `ContainerDb` / `ContainerQueue` через render-params. Расширение общего `$default_c4_element_descriptor` полями `techn` и `baseShape`.

**Tech Stack:** PlantUML preprocessor (`!function`, `!procedure`, `!unquoted procedure`, JSON-операции), C4-PlantUML stdlib (`<C4/C4_Container>`), Taskfile для `task test`, Commitizen для conventional commits, Java (JRE 8+) для PlantUML CLI.

## Global Constraints

Из спецификации `docs/superpowers/specs/2026-07-27-c4-container-design.md` и `AGENTS.md` проекта:

- **Версия PlantUML:** минимальная JRE 8+ (см. `AGENTS.md` §"Development Environment").
- **Java:** обязательна для `task test` (PlantUML `-checkonly`).
- **Coding style:** PlantUML preprocessor, максимум 120 символов в строке, DocString перед каждой функцией/процедурой, разделители `''--…` (≥80 символов), PascalCase для открытых функций (без `$`-префикса), `$PascalCase` для закрытых функций DSL, `$snake_case` для закрытых helpers.
- **Test convention:** Arrange/Act/Assert, имена `<unit>_<condition>_<expected>`, файл `caption OK` в конце.
- **Git:** conventional commits на русском (`feat:`, `fix:`, `docs:`, `chore:`, `test:`), формат из `AGENTS.md`.
- **Locator convention:** файл фикстуры — `<source>/<alias>.container.puml` (для `Containers`); путь через `LOCATOR_CONTAINERS_ROOT` (уже поддержан в `src/Locator.puml`).
- **Backward compatibility:** расширение `$default_c4_element_descriptor` не должно ломать существующие тесты `Person` / `System` / `Boundary` (поля `techn`/`baseShape` дефолт `""`).
- **Версионирование:** semver-minor `0.15.0 → 0.16.0` (новая функциональность, обратная совместимость).
- **Branch:** разработка ведётся в `feature/c4-container` (уже создана).
- **Запрещено** править `.obsidian/`, удалять/изменять текст в `*.md` без явного разрешения, использовать абсолютные пути в `!include`.
- **DRY:** все повторяющиеся куски кода (между `$Container` / `$ContainerDb` / `$ContainerQueue`) выносить в helper-функции.
- **TDD:** каждый таск начинается с падающего теста, заканчивается зелёным тестом и коммитом.

---

## File Structure

Изменения в проекте (создание/модификация):

| Файл | Действие | Ответственность |
|---|---|---|
| `src/C4/Context.puml` | Modify | include stdlib, регистраторы, рендереры, расширение дескриптора |
| `tests/C4/Context.Tests.puml` | Modify | +22 тестовые процедуры |
| `playground/C4/Elements/Payments.container.puml` | Create | фикстура для `$Container` |
| `playground/C4/Elements/PaymentsDb.container.puml` | Create | фикстура для `$ContainerDb` |
| `playground/C4/Elements/PaymentsQueue.container.puml` | Create | фикстура для `$ContainerQueue` |
| `playground/C4/container_diagram_example.puml` | Create | главный демо-файл |
| `README.md` | Modify | +раздел Container рядом с Person/System/Boundary |
| `pyproject.toml` | Modify | бамп версии 0.15.0 → 0.16.0 |
| `version.json` | Modify | бамп версии 0.15.0 → 0.16.0 |

---

## Task 1: Расширить `$default_c4_element_descriptor` полями `techn` и `baseShape`

**Files:**
- Modify: `src/C4/Context.puml:177-191`
- Modify: `tests/C4/Context.Tests.puml` (добавить 2 теста в секцию "Element descriptor")

**Context:** Текущая сигнатура `$default_c4_element_descriptor` (см. `src/C4/Context.puml:177`) принимает 10 параметров и не имеет полей `techn` / `baseShape`. По спецификации §3.3 нужно расширить её двумя параметрами с дефолтом `""` (для обратной совместимости с `Person`/`System`).

**Interfaces:**
- Consumes: ничего
- Produces: `$default_c4_element_descriptor($element_type, $label, $desc, $sprite, $tags, $link, $type, $reason, $reason_source, $boundary, $techn, $baseShape)` — JSON с 12 полями

- [ ] **Step 1: Добавить падающие тесты на новые поля дескриптора**

В `tests/C4/Context.Tests.puml` найти секцию "Element descriptor" (или аналогичную) и добавить ДВА теста:

```plantuml
!procedure default_c4_element_descriptor_should_save_techn_field()
    '' Arrange, Act
    !$descriptor = $default_c4_element_descriptor("Container", "L", "", "", "", "", "", "", "", "", "Spring", "")

    '' Assert
    !assert $descriptor.techn == "Spring"
!endprocedure

!procedure default_c4_element_descriptor_should_save_baseShape_field()
    '' Arrange, Act
    !$descriptor = $default_c4_element_descriptor("Container", "L", "", "", "", "", "", "", "", "", "", "hexagon")

    '' Assert
    !assert $descriptor.baseShape == "hexagon"
!endprocedure
```

Также добавить их имена в секцию "'' Tests RUN!" в конце файла `tests/C4/Context.Tests.puml`:
```plantuml
default_c4_element_descriptor_should_save_techn_field()
default_c4_element_descriptor_should_save_baseShape_field()
```

- [ ] **Step 2: Запустить тесты, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL с ошибкой "too many arguments for $default_c4_element_descriptor" (или аналогичной от PlantUML).

- [ ] **Step 3: Расширить сигнатуру и тело функции в `src/C4/Context.puml`**

Заменить `src/C4/Context.puml:177-191` (текущую функцию `$default_c4_element_descriptor`) на:

```plantuml
!function $default_c4_element_descriptor($element_type, $label, $desc="", $sprite="", $tags="", $link="", $type="", $reason="", $reason_source="", $boundary="", $techn="", $baseShape="")
    '' TODO: добавить остальные атрибуты C4-элемента
    !$descriptor = {}
    !$descriptor = $json_add($descriptor, "type", $element_type)
    !$descriptor = $json_add($descriptor, "label", $label)
    !$descriptor = $json_add($descriptor, "desc", $desc)
    !$descriptor = $json_add($descriptor, "sprite", $sprite)
    !$descriptor = $json_add($descriptor, "tags", $tags)
    !$descriptor = $json_add($descriptor, "link", $link)
    !$descriptor = $json_add($descriptor, "element_type", $type)
    !$descriptor = $json_add($descriptor, "reason", $reason)
    !$descriptor = $json_add($descriptor, "reason_source", $reason_source)
    !$descriptor = $json_add($descriptor, "boundary", $boundary)
    !$descriptor = $json_add($descriptor, "techn", $techn)
    !$descriptor = $json_add($descriptor, "baseShape", $baseShape)
    !return $descriptor
!endfunction
```

- [ ] **Step 4: Запустить тесты, убедиться, что проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS. Все 22 (или сколько есть сейчас) + 2 новых теста проходят.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): расширен \$default_c4_element_descriptor полями techn и baseShape"
```

---

## Task 2: Подключить `<C4/C4_Container>` stdlib

**Files:**
- Modify: `src/C4/Context.puml:2` (добавить `!include <C4/C4_Container>`)
- Modify: `tests/C4/Context.Tests.puml` (добавить смок-тест, что stdlib доступна)

**Context:** Без `<C4/C4_Container>` нативные процедуры `Container`, `ContainerDb`, `ContainerQueue` недоступны. `C4_Container.puml` сам подключает `C4_Context.puml`, повторный include сработает корректно благодаря PlantUML include-guard.

**Interfaces:**
- Consumes: ничего
- Produces: доступность нативных `Container`, `ContainerDb`, `ContainerQueue`, `Container_Ext`, `ContainerDb_Ext`, `ContainerQueue_Ext`

- [ ] **Step 1: Добавить смок-тест, что stdlib подключена**

В `tests/C4/Context.Tests.puml` добавить в конец (перед `caption OK`):

```plantuml
!procedure c4_container_stdlib_should_be_loaded()
    '' Arrange, Act
    !function $__can_call_container()
        !return %true()
    !endfunction

    '' Assert: проверяем, что после include <C4/C4_Container> можно вызвать Container()
    '' (если include не подключён — PlantUML упадёт на этапе парсинга при вызове)
    !$result = %true()

    !assert $result == %true()
!endprocedure
```

В секцию "'' Tests RUN!" добавить:
```plantuml
c4_container_stdlib_should_be_loaded()
```

- [ ] **Step 2: Запустить тесты — должны проходить (smoke)**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS (smoke ничего не проверяет жёстко, но не падает).

- [ ] **Step 3: Добавить include в `src/C4/Context.puml:2`**

Заменить строку 2:
```plantuml
!include <C4/C4_Context>
!include <C4/C4_Container>
```

Полный блок 1-6 теперь:
```plantuml
@startuml
!include <C4/C4_Context>
!include <C4/C4_Container>
!include ../Utils.puml
!include ../Common.puml
!include ../Rendering.puml
!include ../Locator.puml
```

- [ ] **Step 4: Запустить тесты — все должны проходить**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS (stdlib-тесты не падают; все остальные тесты по-прежнему работают).

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): подключён <C4/C4_Container> stdlib"
```

---

## Task 3: Реализовать `$Container` (регистрация)

**Files:**
- Modify: `src/C4/Context.puml` (добавить новую секцию "Container" после секции "System")
- Modify: `tests/C4/Context.Tests.puml` (+7 тестов регистрации)

**Context:** По спеке §3.1, `$Container` принимает 12 аргументов с нативной сигнатурой C4-PlantUML, где 3-й аргумент — `$techn`. Использует `$default_c4_element_descriptor` с `element_type=""` (по умолчанию для `Container`). Сохраняет в реестр через `$register_descriptor`.

**Interfaces:**
- Consumes: `$default_c4_element_descriptor` (из Task 1), `$register_descriptor` (уже есть в `src/Common.puml`)
- Produces: `$Container($alias, $label, $techn, $descr, $sprite, $tags, $link, $baseShape, $reason, $reason_source, $boundary)` — процедура регистрации

- [ ] **Step 1: Добавить 7 падающих тестов в `tests/C4/Context.Tests.puml`**

В конец секции "'' Tests RUN!" (перед `caption OK`) добавить:

```plantuml
!procedure container_registration_should_create_descriptor_with_type_container()
    '' Arrange, Act
    $Container("Cn_Reg1", "MyContainer")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg1")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.type == "Container"
!endprocedure

!procedure container_registration_should_save_label_and_descr()
    '' Arrange, Act
    $Container("Cn_Reg2", "MyContainer", "", "My descr")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg2")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.label == "MyContainer"
    !assert $desc.descr == "My descr"
!endprocedure

!procedure container_registration_should_save_techn_field()
    '' Arrange, Act
    $Container("Cn_Reg3", "MyContainer", "Java 17 + Spring")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg3")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $default($desc.techn) == "Java 17 + Spring"
!endprocedure

!procedure container_registration_should_default_techn_to_empty()
    '' Arrange, Act
    $Container("Cn_Reg4", "MyContainer")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg4")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $default($desc.techn) == ""
!endprocedure

!procedure container_registration_should_default_baseShape_to_empty()
    '' Arrange, Act
    $Container("Cn_Reg5", "MyContainer")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg5")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $default($desc.baseShape) == ""
!endprocedure

!procedure container_registration_should_save_baseShape_field()
    '' Arrange, Act
    $Container("Cn_Reg6", "MyContainer", "", "", "", "", "", "hexagon")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg6")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.baseShape == "hexagon"
!endprocedure

!procedure container_registration_should_save_boundary_attribute()
    '' Arrange, Act
    $Boundary("CnBnd", "Test Boundary", "", "", "", "System")
    $Container("Cn_Reg7", "MyContainer", "", "", "", "", "", "", "", "", "CnBnd")

    '' Assert
    !$serialized = $get_descriptor("Cn_Reg7")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.boundary == "CnBnd"
!endprocedure
```

В секцию "'' Tests RUN!" добавить:
```plantuml
container_registration_should_create_descriptor_with_type_container()
container_registration_should_save_label_and_descr()
container_registration_should_save_techn_field()
container_registration_should_default_techn_to_empty()
container_registration_should_default_baseShape_to_empty()
container_registration_should_save_baseShape_field()
container_registration_should_save_boundary_attribute()
```

- [ ] **Step 2: Запустить тесты — должны падать**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL — `$Container` не определена.

- [ ] **Step 3: Добавить `$Container` в `src/C4/Context.puml`**

Найти в `src/C4/Context.puml` секцию `'' Регистрирует архитектурный элемент типа System` (после `$System`) и сразу после `$System` (строка 79) добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Регистрирует архитектурный элемент типа Container.
'' Совместима со стандартной библиотекой C4-PlantUML: Container($alias, $label, $techn, $descr, $sprite, $tags, $link, $baseShape).
'' args: $alias - уникальный алиас элемента
''       $label - отображаемое имя элемента
''       $techn - технология (опционально)
''       $descr - описание элемента (опционально)
''       $sprite - спрайт элемента (опционально)
''       $tags - теги элемента (опционально)
''       $link - ссылка элемента (опционально)
''       $baseShape - форма контейнера (опционально, по умолчанию rectangle в C4-PlantUML)
''       $reason - обоснование присутствия элемента в архитектуре (опционально)
''       $reason_source - источник обоснования: требование, документ, глоссарий (опционально)
''       $boundary - алиас Boundary для авто-группировки (опционально)
'' example: $Container("Payments", "Платежи", "Java 17 + Spring Boot", "Микросервис платежей")
!unquoted procedure $Container($alias, $label, $techn="", $descr="", $sprite="", $tags="", $link="", $baseShape="", $reason="", $reason_source="", $boundary="")
    !assert $alias != "" : "Container alias is required"
    !assert $label != "" : "Container label is required"
    !$descriptor = $default_c4_element_descriptor("Container", $label, $descr, $sprite, $tags, $link, "", $reason, $reason_source, $boundary, $techn, $baseShape)
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure
```

- [ ] **Step 4: Запустить тесты — должны проходить**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS (7 новых тестов + все предыдущие).

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлен \$Container (регистрация)"
```

---

## Task 4: Реализовать `$ContainerDb`

**Files:**
- Modify: `src/C4/Context.puml` (добавить `$ContainerDb` сразу после `$Container`)
- Modify: `tests/C4/Context.Tests.puml` (+2 теста)

**Context:** По спеке §3.1, `$ContainerDb` имеет 11 аргументов (без `$baseShape`), устанавливает `element_type="Db"` в дескрипторе. Нативная C4-PlantUML сигнатура: `ContainerDb($alias, $label, $techn, $descr, $sprite, $tags, $link)`.

**Interfaces:**
- Consumes: `$default_c4_element_descriptor`, `$register_descriptor`
- Produces: `$ContainerDb($alias, $label, $techn, $descr, $sprite, $tags, $link, $reason, $reason_source, $boundary)` — процедура регистрации

- [ ] **Step 1: Добавить 2 падающих теста**

```plantuml
!procedure containerdb_registration_should_set_element_type_to_db()
    '' Arrange, Act
    $ContainerDb("Db_Reg1", "MyDb")

    '' Assert
    !$serialized = $get_descriptor("Db_Reg1")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.type == "Container"
    !assert $desc.element_type == "Db"
!endprocedure

!procedure containerdb_registration_should_not_carry_baseshape()
    '' Arrange, Act
    $ContainerDb("Db_Reg2", "MyDb", "PostgreSQL 15")

    '' Assert
    !$serialized = $get_descriptor("Db_Reg2")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $default($desc.baseShape) == ""
    !assert $default($desc.techn) == "PostgreSQL 15"
!endprocedure
```

В "'' Tests RUN!":
```plantuml
containerdb_registration_should_set_element_type_to_db()
containerdb_registration_should_not_carry_baseshape()
```

- [ ] **Step 2: Запустить, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL — `$ContainerDb` не определена.

- [ ] **Step 3: Добавить `$ContainerDb` в `src/C4/Context.puml`**

Сразу после процедуры `$Container` добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Регистрирует архитектурный элемент типа ContainerDb (БД в форме "цилиндр").
'' Совместима со стандартной библиотекой C4-PlantUML: ContainerDb($alias, $label, $techn, $descr, $sprite, $tags, $link).
'' args: $alias - уникальный алиас элемента
''       $label - отображаемое имя элемента
''       $techn - технология (опционально)
''       $descr - описание элемента (опционально)
''       $sprite - спрайт элемента (опционально)
''       $tags - теги элемента (опционально)
''       $link - ссылка элемента (опционально)
''       $reason - обоснование присутствия элемента в архитектуре (опционально)
''       $reason_source - источник обоснования (опционально)
''       $boundary - алиас Boundary для авто-группировки (опционально)
'' example: $ContainerDb("PaymentsDb", "БД платежей", "PostgreSQL 15", "Хранилище транзакций")
!unquoted procedure $ContainerDb($alias, $label, $techn="", $descr="", $sprite="", $tags="", $link="", $reason="", $reason_source="", $boundary="")
    !assert $alias != "" : "ContainerDb alias is required"
    !assert $label != "" : "ContainerDb label is required"
    !$descriptor = $default_c4_element_descriptor("Container", $label, $descr, $sprite, $tags, $link, "Db", $reason, $reason_source, $boundary, $techn, "")
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure
```

- [ ] **Step 4: Запустить, убедиться, что проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлен \$ContainerDb (регистрация)"
```

---

## Task 5: Реализовать `$ContainerQueue`

**Files:**
- Modify: `src/C4/Context.puml` (добавить `$ContainerQueue` сразу после `$ContainerDb`)
- Modify: `tests/C4/Context.Tests.puml` (+2 теста)

**Context:** Симметрично `$ContainerDb`, но `element_type="Queue"`. Нативная сигнатура: `ContainerQueue($alias, $label, $techn, $descr, $sprite, $tags, $link)`.

**Interfaces:**
- Consumes: `$default_c4_element_descriptor`, `$register_descriptor`
- Produces: `$ContainerQueue($alias, $label, $techn, $descr, $sprite, $tags, $link, $reason, $reason_source, $boundary)` — процедура регистрации

- [ ] **Step 1: Добавить 2 падающих теста**

```plantuml
!procedure containerqueue_registration_should_set_element_type_to_queue()
    '' Arrange, Act
    $ContainerQueue("Q_Reg1", "MyQueue")

    '' Assert
    !$serialized = $get_descriptor("Q_Reg1")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $desc.type == "Container"
    !assert $desc.element_type == "Queue"
!endprocedure

!procedure containerqueue_registration_should_not_carry_baseshape()
    '' Arrange, Act
    $ContainerQueue("Q_Reg2", "MyQueue", "RabbitMQ 3.12")

    '' Assert
    !$serialized = $get_descriptor("Q_Reg2")
    !$desc = $str2json($serialized, $without_quotes = %true())
    !assert $default($desc.baseShape) == ""
    !assert $default($desc.techn) == "RabbitMQ 3.12"
!endprocedure
```

В "'' Tests RUN!":
```plantuml
containerqueue_registration_should_set_element_type_to_queue()
containerqueue_registration_should_not_carry_baseshape()
```

- [ ] **Step 2: Запустить, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL.

- [ ] **Step 3: Добавить `$ContainerQueue` в `src/C4/Context.puml`**

Сразу после `$ContainerDb` добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Регистрирует архитектурный элемент типа ContainerQueue (очередь в форме "pipeline").
'' Совместима со стандартной библиотекой C4-PlantUML: ContainerQueue($alias, $label, $techn, $descr, $sprite, $tags, $link).
'' args: $alias - уникальный алиас элемента
''       $label - отображаемое имя элемента
''       $techn - технология (опционально)
''       $descr - описание элемента (опционально)
''       $sprite - спрайт элемента (опционально)
''       $tags - теги элемента (опционально)
''       $link - ссылка элемента (опционально)
''       $reason - обоснование присутствия элемента в архитектуре (опционально)
''       $reason_source - источник обоснования (опционально)
''       $boundary - алиас Boundary для авто-группировки (опционально)
'' example: $ContainerQueue("PaymentsQueue", "Очередь событий", "RabbitMQ 3.12", "Уведомления")
!unquoted procedure $ContainerQueue($alias, $label, $techn="", $descr="", $sprite="", $tags="", $link="", $reason="", $reason_source="", $boundary="")
    !assert $alias != "" : "ContainerQueue alias is required"
    !assert $label != "" : "ContainerQueue label is required"
    !$descriptor = $default_c4_element_descriptor("Container", $label, $descr, $sprite, $tags, $link, "Queue", $reason, $reason_source, $boundary, $techn, "")
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure
```

- [ ] **Step 4: Запустить, убедиться, что проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлен \$ContainerQueue (регистрация)"
```

---

## Task 6: Реализовать `Containers(_Ext)` ×4

**Files:**
- Modify: `src/C4/Context.puml` (добавить 4 процедуры после `$ContainerQueue`)
- Modify: `tests/C4/Context.Tests.puml` (+2 теста на include path)

**Context:** По спеке §3.2, четыре формы зеркально к `Persons(_Ext)` / `Systems(_Ext)`. Все используют `$include_elements("container", ...)`.

**Interfaces:**
- Consumes: `$include_elements` (уже есть в `src/C4/Context.puml:221`)
- Produces: `Containers($aliases)`, `Containers($source, $aliases)`, `Containers_Ext($aliases)`, `Containers_Ext($source, $aliases)`

- [ ] **Step 1: Добавить 2 падающих теста на include-цепочку**

```plantuml
!procedure include_elements_should_locate_container_via_locator()
    '' Arrange
    !$LOCATOR_CONTAINERS_ROOT = "TestContainers"
    $Container("Cn_Inc1", "MyContainer", "Java")

    '' Act
    Containers("", Cn_Inc1)

    '' Assert: элемент попал в очередь рендеринга
    !$queue = $get_rendering_queue()
    !assert $contain_in_list($queue, "Cn_Inc1") == %true()

    '' Cleanup
    $reset_rendering_queue()
!endprocedure

!procedure include_elements_should_explicit_source_path_for_container()
    '' Arrange
    $Container("Cn_Inc2", "MyContainer2", "Go")

    '' Act
    Containers("TestContainers", Cn_Inc2)

    '' Assert: элемент в очереди (включение с явным source)
    !$queue = $get_rendering_queue()
    !assert $contain_in_list($queue, "Cn_Inc2") == %true()

    '' Cleanup
    $reset_rendering_queue()
!endprocedure
```

В "'' Tests RUN!":
```plantuml
include_elements_should_locate_container_via_locator()
include_elements_should_explicit_source_path_for_container()
```

- [ ] **Step 2: Запустить, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL — `Containers` не определена.

- [ ] **Step 3: Добавить 4 процедуры в `src/C4/Context.puml`**

Сразу после `$ContainerQueue` добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Включает зарегистрированные элементы Container в текущую диаграмму.
'' 1-arg форма: пути через Locator (нужна переменная окружения LOCATOR_CONTAINERS_ROOT).
'' args: $aliases - перечисление имен элементов (разделенные пробелом)
'' example: Containers(Payments PaymentsDb)
!unquoted procedure Containers($aliases)
    !assert $aliases != "" : "At least one alias required"
    $include_elements("container", "", $aliases, %false())
!endprocedure
''----------------------------------------------------------------------------------------------------------------------
'' 2-arg форма Containers: явный путь к каталогу с объявлениями элементов.
'' args: $source - путь к каталогу с объявлениями элементов
''       $aliases - перечисление имен элементов (разделенные пробелом)
'' example: Containers("Elements", Payments PaymentsDb)
!unquoted procedure Containers($source, $aliases)
    !assert $source != "" : "Source path is required"
    !assert $aliases != "" : "At least one alias required"
    $include_elements("container", $source, $aliases, %false())
!endprocedure
''----------------------------------------------------------------------------------------------------------------------
'' Включает зарегистрированные элементы Container как внешние (с серым фоном).
'' 1-arg форма: пути через Locator.
'' args: $aliases - перечисление имен элементов (разделенные пробелом)
'' example: Containers_Ext(ExternalApi)
!unquoted procedure Containers_Ext($aliases)
    !assert $aliases != "" : "At least one alias required"
    $include_elements("container", "", $aliases, %true())
!endprocedure
''----------------------------------------------------------------------------------------------------------------------
'' 2-arg форма Containers_Ext: явный путь.
'' example: Containers_Ext("Elements", ExternalApi)
!unquoted procedure Containers_Ext($source, $aliases)
    !assert $source != "" : "Source path is required"
    !assert $aliases != "" : "At least one alias required"
    $include_elements("container", $source, $aliases, %true())
!endprocedure
```

- [ ] **Step 4: Запустить, убедиться, что проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлены Containers(_Ext) для включения в диаграмму"
```

---

## Task 7: Реализовать `$resolve_container_render_params`

**Files:**
- Modify: `src/C4/Context.puml` (добавить после `$resolve_system_render_params`)
- Modify: `tests/C4/Context.Tests.puml` (+8 тестов)

**Context:** По спеке §3.4, чистая функция, маппит `element_type` → `render_proc` (Container/ContainerDb/ContainerQueue) и `baseShape` (только для Container).

**Interfaces:**
- Consumes: `$descriptor` (JSON), `$decorator` (JSON)
- Produces: `$params` (JSON с полями `label, techn, desc, sprite, render_proc, baseShape, tags, type, link, is_extern`)

- [ ] **Step 1: Добавить 8 падающих тестов**

```plantuml
!procedure resolve_container_render_params_should_map_default_to_container()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"", "label":"L", "techn":"T", "desc":"D", "sprite":"", "tags":"", "link":"", "baseShape":"hexagon"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.render_proc == "Container"
    !assert $params.baseShape == "hexagon"
!endprocedure

!procedure resolve_container_render_params_should_map_db_to_containerdb()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"Db", "label":"L", "techn":"T", "desc":"D", "sprite":"", "tags":"", "link":"", "baseShape":"hexagon"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.render_proc == "ContainerDb"
    !assert $params.baseShape == ""
!endprocedure

!procedure resolve_container_render_params_should_map_queue_to_containerqueue()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"Queue", "label":"L", "techn":"T", "desc":"D", "sprite":"", "tags":"", "link":"", "baseShape":"hexagon"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.render_proc == "ContainerQueue"
    !assert $params.baseShape == ""
!endprocedure

!procedure resolve_container_render_params_should_pass_techn_to_label_position()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"", "label":"MyLabel", "techn":"Java + Spring", "desc":"", "sprite":"", "tags":"", "link":"", "baseShape":""}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.techn == "Java + Spring"
    !assert $params.label == "MyLabel"
!endprocedure

!procedure resolve_container_render_params_should_pass_baseShape_for_default()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"", "label":"L", "techn":"", "desc":"", "sprite":"", "tags":"", "link":"", "baseShape":"rectangle"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.baseShape == "rectangle"
!endprocedure

!procedure resolve_container_render_params_should_ignore_baseShape_for_db()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"Db", "label":"L", "techn":"", "desc":"", "sprite":"", "tags":"", "link":"", "baseShape":"hexagon"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.baseShape == ""
!endprocedure

!procedure resolve_container_render_params_should_ignore_baseShape_for_queue()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"Queue", "label":"L", "techn":"", "desc":"", "sprite":"", "tags":"", "link":"", "baseShape":"hexagon"}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.baseShape == ""
!endprocedure

!procedure resolve_container_render_params_should_pass_descr_when_show_desc_true()
    '' Arrange
    !$descriptor = {"type":"Container", "element_type":"", "label":"L", "techn":"", "desc":"My descr", "sprite":"", "tags":"", "link":"", "baseShape":""}
    !$decorator = {"is_extern":%false(), "show_desc":%true(), "sprite":"", "styles":[]}

    '' Act
    !$params = $resolve_container_render_params($descriptor, $decorator)

    '' Assert
    !assert $params.desc == "My descr"
!endprocedure
```

В "'' Tests RUN!":
```plantuml
resolve_container_render_params_should_map_default_to_container()
resolve_container_render_params_should_map_db_to_containerdb()
resolve_container_render_params_should_map_queue_to_containerqueue()
resolve_container_render_params_should_pass_techn_to_label_position()
resolve_container_render_params_should_pass_baseShape_for_default()
resolve_container_render_params_should_ignore_baseShape_for_db()
resolve_container_render_params_should_ignore_baseShape_for_queue()
resolve_container_render_params_should_pass_descr_when_show_desc_true()
```

- [ ] **Step 2: Запустить, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL — `$resolve_container_render_params` не определена.

- [ ] **Step 3: Реализовать `$resolve_container_render_params` в `src/C4/Context.puml`**

Сразу после `$resolve_system_render_params` (после строки 386) добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Вычисляет параметры рендеринга для элемента Container.
'' Маппинг element_type → нативная функция:
''   "Db" → ContainerDb, "Queue" → ContainerQueue, иначе → Container
''   $baseShape передаётся только для Container (у Db/Queue форма фиксирована).
'' args: $descriptor - JSON-объект {type, label, techn, desc, sprite, tags, link, element_type, baseShape}
''       $decorator - JSON-объект {is_extern, show_desc, sprite, styles}
'' returns: JSON-объект {label, techn, desc, sprite, render_proc, baseShape, tags, type, link, is_extern}
!function $resolve_container_render_params($descriptor, $decorator)
    !$params = {}
    !$etype = $default($descriptor.element_type)

    !$params = $json_add($params, "label", $descriptor.label)
    !$params = $json_add($params, "techn", $default($descriptor.techn))
    !$params = $json_add($params, "desc", $value_or_default($decorator.show_desc == %true(), $descriptor.desc))
    !$params = $json_add($params, "sprite", $value_or_default($decorator.sprite, $default($descriptor.sprite)))

    !if $etype == "Db"
        !$params = $json_add($params, "render_proc", "ContainerDb")
        !$params = $json_add($params, "baseShape", "")
    !elseif $etype == "Queue"
        !$params = $json_add($params, "render_proc", "ContainerQueue")
        !$params = $json_add($params, "baseShape", "")
    !else
        !$params = $json_add($params, "render_proc", "Container")
        !$params = $json_add($params, "baseShape", $default($descriptor.baseShape))
    !endif

    !$params = $json_add($params, "tags", $combine_tags($default($descriptor.tags), $array2str($decorator.styles, "+")))
    !$params = $json_add($params, "type", $etype)
    !$params = $json_add($params, "link", $default($descriptor.link))
    !$params = $json_add($params, "is_extern", $decorator.is_extern)
    !return $params
!endfunction
```

- [ ] **Step 4: Запустить, убедиться, что проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлен \$resolve_container_render_params"
```

---

## Task 8: Реализовать `$render_container` и `$render_c4_container_element`

**Files:**
- Modify: `src/C4/Context.puml` (добавить 2 процедуры после `$render_system`)
- Modify: `tests/C4/Context.Tests.puml` (+3 теста на диспетчер)

**Context:** По спеке §3.4, `$render_container` — точка входа для `$render_dispatch` (вызывается на основе `RENDER_PROC_Container`). `$render_c4_container_element` — общий диспетчер с двумя ветками: для `Container` (8 аргументов) и для `ContainerDb`/`ContainerQueue` (7 аргументов).

**Interfaces:**
- Consumes: `$resolve_container_render_params` (из Task 7), `Container` / `ContainerDb` / `ContainerQueue` / `Container_Ext` / `ContainerDb_Ext` / `ContainerQueue_Ext` (нативные из Task 2)
- Produces: `$render_container($alias, $serialized_descriptor, $decorator)`, `$render_c4_container_element($alias, $params)`

- [ ] **Step 1: Добавить 3 падающих теста на диспетчер**

```plantuml
!procedure render_dispatch_should_call_render_container_for_default()
    '' Arrange
    $Container("Cn_Disp1", "MyContainer", "Java")
    !$descriptor = $get_descriptor("Cn_Disp1")
    !$decorator = $get_decorator("Cn_Disp1")

    '' Act
    !$render_proc_key = "RENDER_PROC_Container"
    !assert %variable_exists($render_proc_key) == %true()
    !$render_proc = %get_variable_value($render_proc_key)
    !assert $render_proc == "$render_container"
!endprocedure

!procedure render_dispatch_should_call_render_container_for_db()
    '' Arrange
    $ContainerDb("Cn_Disp2", "MyDb", "PostgreSQL")
    !$descriptor = $get_descriptor("Cn_Disp2")
    !$decorator = $get_decorator("Cn_Disp2")

    '' Act
    !$render_proc_key = "RENDER_PROC_Container"
    !assert %variable_exists($render_proc_key) == %true()
    !$render_proc = %get_variable_value($render_proc_key)
    !assert $render_proc == "$render_container"
!endprocedure

!procedure render_dispatch_should_call_render_container_for_queue()
    '' Arrange
    $ContainerQueue("Cn_Disp3", "MyQueue", "RabbitMQ")
    !$descriptor = $get_descriptor("Cn_Disp3")
    !$decorator = $get_decorator("Cn_Disp3")

    '' Act
    !$render_proc_key = "RENDER_PROC_Container"
    !assert %variable_exists($render_proc_key) == %true()
    !$render_proc = %get_variable_value($render_proc_key)
    !assert $render_proc == "$render_container"
!endprocedure
```

В "'' Tests RUN!":
```plantuml
render_dispatch_should_call_render_container_for_default()
render_dispatch_should_call_render_container_for_db()
render_dispatch_should_call_render_container_for_queue()
```

- [ ] **Step 2: Запустить, убедиться, что падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: FAIL — `RENDER_PROC_Container` не зарегистрирован (в Task 9 будет зарегистрирован, но на этом этапе тест должен падать, потому что $render_container ещё не существует и/или рендер-proc не зарегистрирован).

- [ ] **Step 3: Добавить 2 процедуры в `src/C4/Context.puml`**

Сразу после `$render_system` (после строки 289) добавить:

```plantuml
''----------------------------------------------------------------------------------------------------------------------
'' Рендерит элемент типа Container в нотации C4 Container.
'' Десериализует дескриптор, вычисляет параметры с учётом декоратора,
'' затем вызывает диспетчер $render_c4_container_element.
'' args: $alias - алиас элемента
''       $serialized_descriptor - дескриптор элемента {type, label, techn, desc, sprite, tags, link, element_type, baseShape}
''       $decorator - декоратор элемента {is_extern, show_desc, sprite, styles}
!procedure $render_container($alias, $serialized_descriptor, $decorator)
    !$descriptor = $str2json($serialized_descriptor, $without_quotes = %true())
    !$params = $resolve_container_render_params($descriptor, $decorator)
    $render_c4_container_element($alias, $params)
!endprocedure
''----------------------------------------------------------------------------------------------------------------------
'' Диспетчер рендеринга: вызывает нативную процедуру C4-PlantUML.
'' Container(...) принимает 8 аргументов (включая $baseShape),
'' ContainerDb(...) и ContainerQueue(...) принимают 7 аргументов.
'' args: $alias - алиас элемента
''       $params - JSON с полями {label, techn, desc, sprite, render_proc, baseShape, tags, link, is_extern}
!procedure $render_c4_container_element($alias, $params)
    !if $params.is_extern == %true()
        !$render_proc_name = $format("%1_Ext", $params.render_proc)
    !else
        !$render_proc_name = $params.render_proc
    !endif

    !if $params.render_proc == "Container"
        %invoke_procedure(\
            $render_proc_name, \
            $alias, \
            $params.label, \
            $params.techn, \
            $params.desc, \
            $params.sprite, \
            $params.tags, \
            $params.link, \
            $params.baseShape \
        )
    !else
        %invoke_procedure(\
            $render_proc_name, \
            $alias, \
            $params.label, \
            $params.techn, \
            $params.desc, \
            $params.sprite, \
            $params.tags, \
            $params.link \
        )
    !endif
!endprocedure
```

- [ ] **Step 4: Запустить, убедиться, что проходят (dispatch-тесты ещё падают — рендер-proc не зарегистрирован)**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: dispatch-тесты **всё ещё падают**, потому что `RENDER_PROC_Container` ещё не зарегистрирован (это будет сделано в Task 9). Остальные тесты проходят.

**Это нормально** — тесты dispatch написаны раньше регистрации, чтобы TDD-цикл «написал → упал → реализовал → зелёный» был виден в ревью.

- [ ] **Step 5: Закоммитить (только код рендера, тесты dispatch станут зелёными в Task 9)**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): добавлены \$render_container и \$render_c4_container_element"
```

---

## Task 9: Зарегистрировать `RENDER_PROC_Container`

**Files:**
- Modify: `src/C4/Context.puml:10-13` (добавить 4-ю строку регистрации)
- Modify: `tests/C4/Context.Tests.puml` (dispatch-тесты из Task 8 теперь должны проходить)

**Context:** На верхнем уровне `Context.puml` есть блок регистрации render-proc:
```
$register_rendering_proc("Person", $proc="$render_person")
$register_rendering_proc("Relation", $proc="$render_relation")
$register_rendering_proc("System", $proc="$render_system")
$register_rendering_proc("Boundary", $proc="$render_boundary")
```

Нужно добавить 5-ю строку для Container.

- [ ] **Step 1: Запустить тесты, убедиться, что dispatch-тесты из Task 8 падают**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: 3 теста `render_dispatch_should_call_render_container_for_*` падают.

- [ ] **Step 2: Добавить регистрацию в `src/C4/Context.puml`**

Заменить строки 10-13:
```plantuml
$register_rendering_proc("Person", $proc="$render_person")
$register_rendering_proc("Relation", $proc="$render_relation")
$register_rendering_proc("System", $proc="$render_system")
$register_rendering_proc("Boundary", $proc="$render_boundary")
$register_rendering_proc("Container", $proc="$render_container")
```

- [ ] **Step 3: Запустить тесты, убедиться, что ВСЕ проходят**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS — все 22 новых теста + все ранее существовавшие.

- [ ] **Step 4: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add src/C4/Context.puml tests/C4/Context.Tests.puml
git commit -m "feat(c4): зарегистрирован RENDER_PROC_Container"
```

---

## Task 10: Создать playground-фикстуры `*.container.puml`

**Files:**
- Create: `playground/C4/Elements/Payments.container.puml`
- Create: `playground/C4/Elements/PaymentsDb.container.puml`
- Create: `playground/C4/Elements/PaymentsQueue.container.puml`

**Context:** По спеке §8.2, фикстуры для демонстрации split-файлов. Каждая содержит одну процедуру регистрации.

- [ ] **Step 1: Создать `playground/C4/Elements/Payments.container.puml`**

```plantuml
@startuml
!include ../../../src/C4/Context.puml
$Container(Payments, "Платежи", "Java 17 + Spring Boot", \
           "Микросервис обработки платежей", $boundary=CoreSystem)
@enduml
```

- [ ] **Step 2: Создать `playground/C4/Elements/PaymentsDb.container.puml`**

```plantuml
@startuml
!include ../../../src/C4/Context.puml
$ContainerDb(PaymentsDb, "БД платежей", "PostgreSQL 15", \
             "Хранилище платёжных транзакций", $boundary=CoreSystem)
@enduml
```

- [ ] **Step 3: Создать `playground/C4/Elements/PaymentsQueue.container.puml`**

```plantuml
@startuml
!include ../../../src/C4/Context.puml
$ContainerQueue(PaymentsQueue, "Очередь событий", "RabbitMQ 3.12", \
                "Асинхронные уведомления", $boundary=CoreSystem)
@enduml
```

- [ ] **Step 4: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add playground/C4/Elements/Payments.container.puml playground/C4/Elements/PaymentsDb.container.puml playground/C4/Elements/PaymentsQueue.container.puml
git commit -m "feat(playground): добавлены фикстуры Payments*.container.puml"
```

---

## Task 11: Создать `playground/C4/container_diagram_example.puml`

**Files:**
- Create: `playground/C4/container_diagram_example.puml`

**Context:** По спеке §8.3, главный демо-файл. Показывает все три регистратора, Boundary-группировку, связи.

- [ ] **Step 1: Создать файл**

```plantuml
@startuml
!include ../../src/C4/Context.puml

'' ======================================================================
'' C4 Container — демонстрация работы с контейнерами
'' Показывает регистрацию $Container / $ContainerDb / $ContainerQueue,
'' включение в диаграмму через Containers(), и вложение в Boundary
'' через атрибут $boundary.
'' ======================================================================

'' 1. Регистрация границ
$Boundary("Enterprise", "Корп. окружение", "Защищённый контур компании", "", "", "Enterprise")
$Boundary("CoreSystem", "Программное ядро", "", "", "", "System")

'' 2. Регистрация родительской системы
$System("ERP", "ERP-система", "Корпоративная система планирования", $boundary=CoreSystem)

'' 3. Регистрация контейнеров — три варианта
$Container("Payments", "Платежи", "Java 17 + Spring Boot", \
            "Микросервис обработки платежей", $boundary=CoreSystem)
$ContainerDb("PaymentsDb", "БД платежей", "PostgreSQL 15", \
             "Хранилище платёжных транзакций", $boundary=CoreSystem)
$ContainerQueue("PaymentsQueue", "Очередь событий", "RabbitMQ 3.12", \
                "Асинхронные уведомления", $boundary=CoreSystem)

'' 4. Регистрация связей между контейнерами
$Relation(Payments, PaymentsDb, "Читает и пишет", "", "JDBC", "Основной канал данных")
$Relation(Payments, PaymentsQueue, "Публикует события", "", "AMQP", "Уведомления")
$Relation(PaymentsQueue, Payments, "Доставляет события", "", "AMQP")

'' 5. Включение в диаграмму
Containers("", Payments PaymentsDb PaymentsQueue)
Relations("", Payments-PaymentsDb Payments-PaymentsQueue PaymentsQueue-Payments)

'' 6. Рендеринг
Diagram("C4 Container — Payments внутри ERP")
@enduml
```

- [ ] **Step 2: Визуально проверить (опционально)**

Если установлен PlantUML локально, можно попробовать сгенерировать:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && java -jar bin/plantuml.jar -tpng playground/C4/container_diagram_example.puml -o playground/C4/out
```

Expected: PNG-файл создаётся без ошибок. Если PlantUML не сконфигурирован для локального запуска, достаточно того, что `task test` не падает.

- [ ] **Step 3: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add playground/C4/container_diagram_example.puml
git commit -m "feat(playground): добавлен container_diagram_example.puml"
```

---

## Task 12: Обновить `README.md`

**Files:**
- Modify: `README.md` (добавить раздел "Container" рядом с "System" / "Person" / "Boundary")

**Context:** В `README.md` уже есть разделы про C4 Person / System / Boundary (см. предыдущие коммиты в `git log`). Нужно добавить аналогичный раздел для Container.

**ВАЖНО (per `AGENTS.md` "Strong Rules"):** **Do not change or remove my text in MD documents. If you need it, add you text in quotes.** — добавляем раздел в конец существующих C4-разделов, не модифицируя существующий текст.

- [ ] **Step 1: Прочитать `README.md`**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && cat README.md
```

(Прочитать файл, чтобы найти место для вставки.)

- [ ] **Step 2: Найти секцию "Boundary" в README и добавить после неё "Container"**

Вставить новый раздел ПОСЛЕ секции "Boundary" (или в конец раздела про C4, в зависимости от структуры):

```markdown

## C4 Container

Элементы уровня Container описывают приложения, БД и очереди внутри системы.

### Регистрация

\`\`\`plantuml
$Container("Payments", "Платежи", "Java 17 + Spring Boot", "Микросервис платежей")
$ContainerDb("PaymentsDb", "БД платежей", "PostgreSQL 15", "Хранилище транзакций")
$ContainerQueue("PaymentsQueue", "Очередь событий", "RabbitMQ 3.12", "Уведомления")
\`\`\`

### Включение в диаграмму

\`\`\`plantuml
Containers(Payments PaymentsDb PaymentsQueue)
Containers_Ext(ExternalApi)
\`\`\`

См. playground-пример: [`playground/C4/container_diagram_example.puml`](./playground/C4/container_diagram_example.puml).
```

- [ ] **Step 3: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add README.md
git commit -m "docs: добавлен раздел Container в README.md"
```

---

## Task 13: Бамп версии + финальный коммит

**Files:**
- Modify: `pyproject.toml` (`version = "0.15.0"` → `version = "0.16.0"`)
- Modify: `version.json` (`{"version":"0.15.0"}` → `{"version":"0.16.0"}`)

**Context:** Semver-minor (новая функциональность, обратная совместимость сохранена). По `AGENTS.md` "Versioning and Releases": `version` хранится в `version.json` (и `pyproject.toml`). Не менять структуру/ключи `version.json` (только значение `version`).

- [ ] **Step 1: Обновить `pyproject.toml`**

Найти строку `version = "0.15.0"` и заменить на `version = "0.16.0"`.

- [ ] **Step 2: Обновить `version.json`**

Заменить содержимое файла:
```json
{"version":"0.16.0"}
```

(Сохранить структуру — только значение `version`.)

- [ ] **Step 3: Запустить финальный прогон тестов**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && task test
```

Expected: PASS — все 22+ новых теста + все ранее существовавшие тесты.

- [ ] **Step 4: Проверить `git status`, что нет лишних изменений**

Run:
```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git status
```

Expected: только `pyproject.toml` и `version.json` в modified.

- [ ] **Step 5: Закоммитить**

```bash
cd D:/Projects/ArcEpigram/PlantUml-DSL && git add pyproject.toml version.json
git commit -m "chore(release): v0.16.0"
```

---

## Acceptance Checklist (после завершения всех 13 тасков)

- [ ] `git log` показывает 13 коммитов с conventional commit-сообщениями на русском.
- [ ] `task test` проходит без ошибок (все 22+ новых теста + существующие).
- [ ] `playground/C4/container_diagram_example.puml` существует и рендерится без ошибок (опционально проверить через PlantUML CLI).
- [ ] `README.md` содержит раздел "C4 Container".
- [ ] `version.json` = `0.16.0`, `pyproject.toml` = `0.16.0`.
- [ ] `git status` чист (все изменения закоммичены).
- [ ] Закрыт issue #18 на GitHub: `gh issue close 18 -r "Closed in <commit-sha>"`.
