# C4 Container — дизайн и спецификация

**Issue:** [#18](https://github.com/ArcEpigram/PlantUml-DSL/issues/18)
**Дата:** 2026-07-27
**Статус:** утверждён, готов к плану реализации
**Версия:** 0.15.0 → 0.16.0 (semver-minor, новая функциональность)

---

## 1. Цель и мотивация

Реализовать в DSL поддержку архитектурного элемента **C4 Container** и его вариантов (`ContainerDb`, `ContainerQueue`) в стиле, единообразном с уже реализованными `Person` / `System` / `Boundary`.

**Зачем:**
- Завершить иерархию C4 (Person → System → Container → Component), покрыть уровень 2 модели C4.
- Дать пользователю возможность описывать контейнеры (приложения, БД, очереди) в декларативном DSL-стиле с дескрипторами, декораторами, Locator, Boundary-группировкой.
- Сохранить совместимость со стандартной библиотекой C4-PlantUML: `Container`, `ContainerDb`, `ContainerQueue` и их `_Ext` варианты.

**Что НЕ делается в этой итерации (YAGNI):**
- C4 Component (issue на будущее).
- Поддержка `$scope` в `Containers(...)` для вложенности в `System`.
- Свои базовые формы (используется `rectangle` по умолчанию из C4-PlantUML).

---

## 2. Архитектура

### 2.1. Текущее состояние (что уже готово)

| Слой | Состояние | Файл |
|---|---|---|
| `$default_c4_element_descriptor` принимает `"Container"` | ✅ | `src/C4/Context.puml:177` |
| `Locator` (контейнер → `LOCATOR_CONTAINERS_ROOT`) | ✅ | `src/Locator.puml:14, 73` |
| `$__plural("container")` → `"Containers"` | ✅ | `src/Locator.puml:51` |
| `$resolve_boundary_native_proc("Container")` → `Container_Boundary` | ✅ | `src/C4/Context.puml:593` |
| `$emit_container_boundary_open` | ✅ | `src/C4/Context.puml:673` |
| `$Boundary(..., $type="Container")` | ✅ | `src/C4/Context.puml:651` |
| **Регистрация `$Container` / `$ContainerDb` / `$ContainerQueue`** | ❌ | — |
| **Рендер `$render_container`** | ❌ | — |
| **Маппинг `element_type` → `ContainerDb`/`ContainerQueue`** | ❌ | — |
| **Тесты Container** | ❌ | — |

### 2.2. Целевая архитектура

```
┌──────────────────────────────────────────────────────────┐
│ src/C4/Context.puml                                       │
│                                                            │
│  ─── Новое: ───────────────────────────────────────────  │
│  !include <C4/C4_Container>                                │
│  $register_rendering_proc("Container", $proc="$render_c") │
│                                                            │
│  $Container(alias, label, $techn, $descr, ...)   [NEW]    │
│  $ContainerDb(alias, label, $techn, $descr, ...)  [NEW]    │
│  $ContainerQueue(alias, label, $techn, $descr,...)[NEW]   │
│  Containers(_Ext) варианты                        [NEW]   │
│  $render_container(...)                          [NEW]    │
│  $resolve_container_render_params(...)           [NEW]    │
│  $render_c4_container_element(...)               [NEW]    │
│                                                            │
│  ─── Изменённое: ──────────────────────────────────────  │
│  $default_c4_element_descriptor +$techn +$baseShape       │
│                                                            │
│  ─── Неизменное: ──────────────────────────────────────  │
│  $include_elements, $render_dispatch, $register_rendering_proc│
│  Locator.puml, Common.puml, Rendering.puml                │
└──────────────────────────────────────────────────────────┘
```

**Принцип:** диспетчер рендеринга (`$render_dispatch`) ищет `RENDER_PROC_Container` → вызывает `$render_container` → `Container(...)` / `ContainerDb(...)` / `ContainerQueue(...)` (нативные из `<C4/C4_Container>`).

---

## 3. Поток данных и контракт API

### 3.1. Регистрация элементов (открытое API)

```plantuml
!unquoted procedure $Container($alias, $label, $techn="", $descr="", \
                                $sprite="", $tags="", $link="", \
                                $baseShape="", $reason="", $reason_source="", \
                                $boundary="")
    !$descriptor = $default_c4_element_descriptor(\
        "Container", $label, $descr, $sprite, $tags, $link, \
        "", $reason, $reason_source, $boundary)
    !$descriptor = $json_add($descriptor, "techn", $techn)
    !$descriptor = $json_add($descriptor, "baseShape", $baseShape)
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure

!unquoted procedure $ContainerDb($alias, $label, $techn="", $descr="", \
                                  $sprite="", $tags="", $link="", \
                                  $reason="", $reason_source="", \
                                  $boundary="")
    !$descriptor = $default_c4_element_descriptor(\
        "Container", $label, $descr, $sprite, $tags, $link, \
        "Db", $reason, $reason_source, $boundary)
    !$descriptor = $json_add($descriptor, "techn", $techn)
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure

!unquoted procedure $ContainerQueue($alias, $label, $techn="", $descr="", \
                                     $sprite="", $tags="", $link="", \
                                     $reason="", $reason_source="", \
                                     $boundary="")
    !$descriptor = $default_c4_element_descriptor(\
        "Container", $label, $descr, $sprite, $tags, $link, \
        "Queue", $reason, $reason_source, $boundary)
    !$descriptor = $json_add($descriptor, "techn", $techn)
    $register_descriptor($alias, $json2str($descriptor, %true()))
!endprocedure
```

**Сигнатуры 1:1 как в `<C4/C4_Container>` (C4-PlantUML stdlib).** Параметр `$type` в `$Container` отсутствует — тип жёстко задан процедурой (упрощение по сравнению с `$System`, у которого `$type="Db"/"Queue"`).

**Guard clauses (programming-guideline §8):**
- `!assert $alias != "" : "Container alias is required"`
- `!assert $label != "" : "Container label is required"`

### 3.2. Включение в диаграмму (открытое API)

```plantuml
!unquoted procedure Containers($aliases)
    $include_elements("container", "", $aliases, %false())
!endprocedure
!unquoted procedure Containers($source, $aliases)
    $include_elements("container", $source, $aliases, %false())
!endprocedure
!unquoted procedure Containers_Ext($aliases)
    $include_elements("container", "", $aliases, %true())
!endprocedure
!unquoted procedure Containers_Ext($source, $aliases)
    $include_elements("container", $source, $aliases, %true())
!endprocedure
```

Зеркально к `Persons(_Ext)` / `Systems(_Ext)`. Никакой новой логики — `$include_elements` уже принимает произвольный `element_type`.

Файл-фикстура: `*.container.puml` — единый формат для всех трёх регистраторов (внутри `Db.container.puml` вызывается `$ContainerDb(...)`).

### 3.3. Изменения в `$default_c4_element_descriptor`

Добавить два параметра с дефолтом `""`:
- `$techn` (для Container)
- `$baseShape` (для Container)

```plantuml
!function $default_c4_element_descriptor($element_type, $label, $desc="", $sprite="", $tags="", \
                                          $link="", $type="", $reason="", $reason_source="", \
                                          $boundary="", $techn="", $baseShape="")
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

**Совместимость:** для `$Person` и `$System` поля `techn` и `baseShape` будут присутствовать (пустые), но их рендереры эти поля не читают → поведение не меняется. Все существующие тесты продолжают работать.

### 3.4. Рендеринг

**Регистрация рендер-proc (на верхнем уровне `Context.puml`):**
```plantuml
$register_rendering_proc("Container", $proc="$render_container")
```

**Точка входа:**
```plantuml
!procedure $render_container($alias, $serialized_descriptor, $decorator)
    !$descriptor = $str2json($serialized_descriptor, $without_quotes = %true())
    !$params = $resolve_container_render_params($descriptor, $decorator)
    $render_c4_container_element($alias, $params)
!endprocedure
```

**Вычисление параметров рендеринга (чистая функция):**
```plantuml
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

**Маппинг `element_type` → `render_proc` и `baseShape`:**

| `descriptor.element_type` | `params.render_proc` | `params.baseShape` |
|---|---|---|
| `"Db"` | `ContainerDb` | `""` (игнор) |
| `"Queue"` | `ContainerQueue` | `""` (игнор) |
| `""` или иное | `Container` | `$default($descriptor.baseShape)` |

**Диспетчер рендера (вызывает нативные процедуры):**
```plantuml
!procedure $render_c4_container_element($alias, $params)
    !if $params.is_extern == %true()
        !$render_proc_name = $format("%1_Ext", $params.render_proc)
    !else
        !$render_proc_name = $params.render_proc
    !endif

    !if $params.render_proc == "Container"
        '' Container(...) принимает 8 аргументов, включая $baseShape
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
        '' ContainerDb(...) и ContainerQueue(...) принимают 7 аргументов (без $baseShape)
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

**Конвенция имен:** `Container_Ext`, `ContainerDb_Ext`, `ContainerQueue_Ext` — все определены в `<C4/C4_Container>`, ничего не нужно объявлять вручную.

**Никаких `Render_Container`:** вызываем `Container` / `ContainerDb` / `ContainerQueue` / `Container_Ext` / `ContainerDb_Ext` / `ContainerQueue_Ext` напрямую.

### 3.5. Подключение нативной stdlib

`src/C4/Context.puml:2` — добавить `!include <C4/C4_Container>` сразу после `!include <C4/C4_Context>`. PlantUML include-guard предотвращает двойную загрузку `C4_Context`.

---

## 4. Изменяемые файлы

| Файл | Что меняется |
|---|---|
| `src/C4/Context.puml` | + `!include <C4/C4_Container>`; + `$register_rendering_proc("Container", ...)`; + `$Container`, `$ContainerDb`, `$ContainerQueue`; + `Containers(_Ext)` ×4; + `$render_container`, `$resolve_container_render_params`, `$render_c4_container_element`; расширить `$default_c4_element_descriptor` параметрами `$techn`, `$baseShape` |
| `tests/C4/Context.Tests.puml` | + секции «Container registration», «Container render params», «Container render dispatch» |
| `playground/C4/Elements/Payments.container.puml` | NEW: фикстура для `$Container` |
| `playground/C4/Elements/PaymentsDb.container.puml` | NEW: фикстура для `$ContainerDb` |
| `playground/C4/Elements/PaymentsQueue.container.puml` | NEW: фикстура для `$ContainerQueue` |
| `playground/C4/container_diagram_example.puml` | NEW: демонстрация C4 Container (вариант A — рядом с `context_with_bounds_example.puml`) |
| `README.md` | + раздел «Container» рядом с «Person» / «System» / «Boundary» |
| `pyproject.toml` | бамп версии `0.15.0 → 0.16.0` |
| `version.json` | бамп версии `0.15.0 → 0.16.0` |

---

## 5. Тестирование

### 5.1. Unit-тесты в `tests/C4/Context.Tests.puml`

Все процедуры — в стиле Arrange / Act / Assert, именование `<unit>_<condition>_<expected>`.

**Container registration:**
1. `container_registration_should_create_descriptor_with_type_container`
2. `container_registration_should_save_label_and_descr`
3. `container_registration_should_save_techn_field`
4. `container_registration_should_default_techn_to_empty`
5. `container_registration_should_default_baseShape_to_empty`
6. `container_registration_should_save_baseShape_field`
7. `container_registration_should_save_boundary_attribute`

**ContainerDb registration:**
8. `containerdb_registration_should_set_element_type_to_db`
9. `containerdb_registration_should_not_carry_baseshape`

**ContainerQueue registration:**
10. `containerqueue_registration_should_set_element_type_to_queue`
11. `containerqueue_registration_should_not_carry_baseshape`

**Container render params:**
12. `resolve_container_render_params_should_map_default_to_container`
13. `resolve_container_render_params_should_map_db_to_containerdb`
14. `resolve_container_render_params_should_map_queue_to_containerqueue`
15. `resolve_container_render_params_should_pass_techn_to_label_position`
16. `resolve_container_render_params_should_pass_baseShape_for_default`
17. `resolve_container_render_params_should_ignore_baseShape_for_db`
18. `resolve_container_render_params_should_ignore_baseShape_for_queue`
19. `resolve_container_render_params_should_pass_descr_when_show_desc_true`

**Container render dispatch:**
20. `render_dispatch_should_call_render_container_for_default`
21. `render_dispatch_should_call_render_container_for_db`
22. `render_dispatch_should_call_render_container_for_queue`

**Итого:** +22 теста.

### 5.2. Защитные тесты обратной совместимости

Добавить (или подтвердить существующие) тесты на `$Person` / `$System`:
- `person_registration_should_not_carry_techn_field` (или, точнее, поле есть, но пустое).
- `system_registration_should_not_carry_techn_field`.

### 5.3. Запуск тестов

`task test` — обязательно перед коммитом (per `AGENTS.md`).

---

## 6. Обработка ошибок

| Ситуация | Поведение |
|---|---|
| Пустой `$alias` / `$label` в `$Container*` | `!assert` → fail-fast (per `programming-guideline §8`) |
| `$default_c4_element_descriptor` вызван с неизвестным `$element_type` | Без изменений (валидация — на стороне рендерера) |
| `Container(...)` нативный вызов с пустым `$baseShape` | C4-PlantUML использует `rectangle` по умолчанию (поведение stdlib) |
| `Containers(_Ext)` с пустым `$aliases` | Guard: `!assert $aliases != ""` |
| `LOCATOR_CONTAINERS_ROOT` не задан | `!include` пропускается, элемент остаётся в очереди рендеринга (поведение Locator по умолчанию) |

---

## 7. Совместимость и риски

| Риск | Митигация |
|---|---|
| `!include <C4/C4_Container>` дублирует `C4_Context` | PlantUML include-guard, проект уже использует похожие include (`Locator.puml` подключается из тестов) |
| Добавление `$techn`, `$baseShape` в общий дескриптор | Поля дефолтные `""`, рендереры `Person`/`System` не читают — поведение не меняется |
| Сигнатура `$Container` отличается от `$Person`/`$System` (3-й арг `$techn`) | Неизбежно из-за совместимости с C4-PlantUML, документировано в DocString |
| `$Boundary(..., $type="Container")` (issue #13) остаётся как есть | `$Boundary` для **границ**, `$Container` для **элементов** внутри `Container_Boundary` — разные сущности, не конфликтуют |
| `$baseShape` нативно не поддерживается для `ContainerDb`/`ContainerQueue` | `$render_c4_container_element` имеет две ветки: `Container` (8 аргументов) и `ContainerDb`/`ContainerQueue` (7 аргументов). `$baseShape` передаётся только в `Container`. |

---

## 8. Playground-пример

### 8.1. Расположение

**Вариант A:** `playground/C4/container_diagram_example.puml` + `playground/C4/Elements/Payments*.container.puml`. Рядом с `context_with_bounds_example.puml`.

### 8.2. Фикстуры `*.container.puml`

**`playground/C4/Elements/Payments.container.puml`:**
```plantuml
@startuml
!include ../../../src/C4/Context.puml
$Container(Payments, "Платежи", "Java 17 + Spring Boot", \
           "Микросервис обработки платежей", $boundary=CoreSystem)
@enduml
```

**`playground/C4/Elements/PaymentsDb.container.puml`:**
```plantuml
@startuml
!include ../../../src/C4/Context.puml
$ContainerDb(PaymentsDb, "БД платежей", "PostgreSQL 15", \
             "Хранилище платёжных транзакций", $boundary=CoreSystem)
@enduml
```

**`playground/C4/Elements/PaymentsQueue.container.puml`:**
```plantuml
@startuml
!include ../../../src/C4/Context.puml
$ContainerQueue(PaymentsQueue, "Очередь событий", "RabbitMQ 3.12", \
                "Асинхронные уведомления", $boundary=CoreSystem)
@enduml
```

### 8.3. Главный демо-файл `playground/C4/container_diagram_example.puml`

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

### 8.4. Что демонстрирует пример

| Сценарий | Демонстрируемая фича |
|---|---|
| `$Boundary(..., $type="Enterprise")` + `$System(..., $boundary=...)` | Существующая фича (issue #13) |
| `$Container` с `$techn` | Новая фича issue #18 |
| `$ContainerDb`, `$ContainerQueue` | Новая фича (по аналогии с C4 stdlib) |
| `$boundary=CoreSystem` на контейнерах | Авто-группировка в System Boundary |
| `Containers(...)` | Общая команда для всех трёх типов |
| `Relations(...)` между контейнерами | Связи внутри контейнерного уровня C4 |
| `Diagram(...)` | Рендеринг иерархии Enterprise → System → Containers |

---

## 9. Версионирование и коммит

- **Версия:** `0.15.0 → 0.16.0` (semver-minor, новая функциональность, обратная совместимость сохранена).
- **Файлы версий:** `pyproject.toml`, `version.json`.
- **Коммит (conventional commits на русском):**
  ```
  feat: добавлен C4 Container с вариантами ContainerDb и ContainerQueue

  - Три регистратора $Container / $ContainerDb / $ContainerQueue
    с нативными сигнатурами C4-PlantUML (3-й аргумент — $techn).
  - Общая команда Containers(_Ext) для включения в диаграмму.
  - Рендеринг через Render_Container (Container / ContainerDb / ContainerQueue)
    с маппингом element_type → render_proc.
  - Расширен $default_c4_element_descriptor полями $techn и $baseShape.
  - Подключён <C4/C4_Container> stdlib.
  - +22 теста в tests/C4/Context.Tests.puml.
  - Playground-пример container_diagram_example.puml.
  - Бамп версии 0.15.0 → 0.16.0.
  - Закрывает issue #18.
  ```

---

## 10. Что НЕ делаем (YAGNI)

- ❌ C4 Component — отдельный issue на будущее.
- ❌ Поддержка `$scope` в `Containers(...)` для вложенности в `System` — отдельный issue.
- ❌ Свои базовые формы (используется `rectangle` по умолчанию из C4-PlantUML).
- ❌ `Render_Container` / `Render_ContainerDb` / `Render_ContainerQueue` — вызываем нативные процедуры напрямую.
- ❌ `$type` параметр в `$Container` — тип жёстко задан процедурой.
- ❌ `ContainerScope(...)` / `ContainerGroup(...)` — это работа для будущих итераций (как `Group()` для Boundary).
