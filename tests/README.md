# Тесты

## Запуск

```bash
task test   # java -jar bin/plantuml.jar -checkonly tests/*.Tests.puml tests/C4/*.Tests.puml
```

Проверка — это парсинг + выполнение `!assert` в режиме `-checkonly` (без рендеринга).

## Структура

- Один тест-файл на исходный модуль: `src/<Module>.puml` ↔ `tests/<Module>.Tests.puml`.
- Тест подключает модуль через относительный `!include`:
  - `!include ../src/<Module>.puml` (из `tests/`);
  - `!include ../../src/C4/<Module>.puml` (из `tests/C4/`).
- Фикстуры лежат в подкаталогах `Test*` рядом с тестом (например, `tests/C4/TestPersons/`, `tests/TestLocator/`).

## Паттерн теста (Arrange / Act / Assert)

Каждая тестовая процедура следует трёхфазному шаблону:

```plantuml
!procedure <unit>_<condition>_<expected>()
    '' Arrange
    !$desc = {}
    !$desc = $json_add($desc, "type", "Person")

    '' Act
    !$params = $resolve_person_render_params($desc, $decor)

    '' Assert
    !assert $params.label == "Alice"
!endprocedure
```

## Секция запуска

Все тестовые процедуры вызываются по имени в секции `'' Tests RUN!` в конце файла, после чего идёт `caption OK`.

## Соглашения

- Имя теста — snake_case без префикса, паттерн `<unit>_<condition>_<expected>`.
- Проверки — только через `!assert`.
- Фикстуры элементов и связей — «голые» вызовы `$Person(...)` / `$Relation(...)` без `@startuml`/`@enduml` (подключаются через `!include` внутрь теста).
