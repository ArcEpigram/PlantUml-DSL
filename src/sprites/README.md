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

## Авто-подбор спрайта по технологиям

Если у контейнера не задан явный спрайт (через `$sprite=` в дескрипторе или
декораторе), DSL автоматически подбирает пиктограмму по списку его технологий
(сценарий 3.3 issue #22).

За подбор отвечают функции в `src/C4/Technologies.puml`:

- `$get_container_main_tech_from_technologies($technologies)` — выбирает
  наиболее значимую технологию и возвращает её **каноническое имя** из
  справочника (не псевдоним спрайта), т.к. библиотеки пиктограмм могут быть
  любыми внешними.
- `$get_sprite_from_technologies($technologies)` — вызывает первую, затем ищет
  имя спрайта в справочнике; возвращает `""`, если технология неизвестна или
  не имеет стандартного спрайта.

Приоритет категорий при выборе основной технологии:
1. framework / datastore / queue / cache (Spring Boot, PostgreSQL, RabbitMQ, Redis, …)
2. platform (JVM, .NET, Node.js)
3. language (Java, C#, Python, …)

У технологии может быть несколько альтернативных имён (например `spring`,
`spring-boot`, `spring boot` → `Spring Boot`). Справочник MVP охватывает
~18 канонических записей и расширяется в `$__tech_catalog()`.

> **Примечание о PlantUML 1.2026.5:** встроенная `%lower_case` в этой версии
> недоступна, поэтому нормализация регистра реализована посимвольно в
> `$__to_lower()`. Разбор строк (`%splitstr`, `%splitstr_regex`) внутри
> `!function` работает корректно.

## Известные ограничения

- Спрайт задаётся **явно** через параметр `$sprite=` в `$Container(...)`,
  либо через `ApplySprite(...)`, либо **автоматически** по списку технологий
  (см. раздел «Авто-подбор спрайта по технологиям» выше).