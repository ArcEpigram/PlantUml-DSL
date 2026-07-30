# Локальные sprite-обёртки

Каталог `src/sprites/` содержит тонкие обёртки над sprite-библиотеками PlantUML.

## Зачем

DSL не должен привязываться к реализации конкретной sprite-библиотеки. Обёртки
дают пользователю **стабильное имя спрайта** (`$spring-icon`, `$ma_database`,
`$server`), которое не зависит от версии PlantUML и от пути include.

## Содержимое

| Файл | Спрайт | Источник |
|---|---|---|
| `spring-icon.puml` | Spring Boot | `<logos/spring-icon>` |
| `database.puml`    | PostgreSQL/база | `<material/database>` |
| `server.puml`      | RabbitMQ/сервер | `<tupadr3/font-awesome/server>` |

Каждый файл предоставляет функцию-обёртку (`$getSpringIcon`, `$getDatabase`,
`$getServer`), которая нормализует формат алиаса (`<$name>` или `<&name>`).

## Почему обёртки не содержат `!include <stdlib/...>`

PlantUML требует, чтобы stdlib-инклуды шли **в самом верху** диаграммы, до
любых других `!include`. Если обёртка содержит `!include <logos/spring-icon>`,
то подключить её можно будет только в самом начале — до `Context.puml`, иначе
PlantUML выдаёт "Some diagram description contains errors".

Чтобы пользователь сам решал, какие спрайты ему нужны, и не зависел от порядка
include, обёртки в этом каталоге **не выполняют** `!include <...>`. Они только
предоставляют функции-обёртки.

Реальные спрайты пользователь подключает сам в самом верху своей диаграммы:

```plantuml
@startuml
!include <logos/spring-icon>
!include <material/database>
!include <tupadr3/font-awesome/server>
!include <путь/к/dsl>/C4/Context.puml
...
@enduml
```

После этого доступны спрайты `<$spring-icon>`, `<$ma_database>` (или `<$database>` —
зависит от версии PlantUML), `<$server>`.

## Использование в playground/тестах

В playground-примерах и тестах подключение делается явно:

```plantuml
!include <logos/spring-icon>
!include <путь/к/dsl>/C4/Context.puml

Container(api, "API", "Spring Boot", "Backend", $sprite="spring-icon")
```

DSL получает только строку `"spring-icon"`. Реальное наличие спрайта в PlantUML
обеспечивается тем, что пользователь подключил `<logos/spring-icon>` явно.

Если спрайт с указанным именем не подключён, PlantUML всё равно отрендерит
диаграмму, но контейнер будет без пиктограммы.

## Как добавить новую обёртку

1. Скопировать `spring-icon.puml` под новым именем.
2. Заменить имя функции `$getSpringIcon` на новое.
3. Заменить комментарии.
4. Имя спрайта, которое генерирует PlantUML при `!include <...>`, можно увидеть,
   открыв файл из stdlib — он содержит `sprite $<name> ...`. Именно это имя
   и нужно передавать в `$sprite=...`.

## Известные ограничения

- Авто-подбор спрайта по списку технологий (`$get_sprite_from_technologies`)
  **не реализован** из-за ограничений PlantUML 1.2026.5: `%splitstr` и
  `%set_variable_value` некорректно работают с длинными строками внутри
  `!function` / `!procedure`. См. issue #22.
- Спрайт **всегда задаётся явно** через параметр `$sprite=` в
  `$Container(...)` или через `ApplySprite(...)`.