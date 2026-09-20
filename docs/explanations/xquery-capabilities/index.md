# Возможности XQuery

Одна и та же конструкция XQuery в WebSoft HCM может возвращать разные данные
в зависимости от сборки и СУБД. Ни успешная трансляция в SQL, ни отсутствие
исключения не гарантируют правильного результата: могут пропасть переименованные
поля, измениться сторона внешнего соединения или вернуться пустая коллекция.

Ниже — проверенные формы запросов и их результаты на пяти сборках, отдельно
для Microsoft SQL Server и PostgreSQL. Совместимость относится к **приведённому
запросу**, а не ко всем возможным сочетаниям функций с таким названием.

## Условия проверки и обозначения

Проверки выполнены 20 сентября 2026 года на одинаковых данных каталогов.

| Обозначение | Полная версия WebSoft HCM Server |
| --- | --- |
| 434 | `2022.1.3.434` |
| 906 | `2023.2.906` |
| 1132 | `2025.1.1132` |
| 1333 | `2025.1.1333` |
| 1525 | `2025.2.1525` |

Во всех прогонах `XQueryTemplateCache=False`. Результаты не подтверждают работу
тех же запросов с включённым шаблонным кэшем. Для полнотекстового поиска включён
`LuceneFTIndex=True`; также использовались `DateFormat=dmy`,
`PreventNonIndexOrder=False`. В конфигурациях 434, 1132, 1333 и 1525 явно указаны
`IndexOptimizerMode=None`, `InnerJoin=True`; в снимках 906 эти два параметра
не заданы, и их значения по умолчанию здесь не утверждаются.
Автоматически добавленная сортировка SQL рассматривается при этих настройках;
эффект их изменения не проверялся.

В каждой ячейке таблиц ниже два результата: **MSSQL / PostgreSQL**.

| Знак | Значение |
| --- | --- |
| **<abbr title="Результат совпал с ожиданием">✓</abbr>** | Совпали ожидаемые строки, проверяемые поля и значения конечного результата. |
| **<abbr title="Результат отличается от ожидаемого">≠</abbr>** | Запрос вернул результат, но он отличается от ожидаемого: неверные строки, пустая коллекция или отсутствующие поля. |
| **<abbr title="Ошибка выполнения запроса">О</abbr>** | Ошибка выполнения, включая ошибку SQL при чтении возвращённой коллекции. |
| **<abbr title="JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr>** | Для внешних JOIN на 434: принято ограничение поддержки по предыдущим проверкам; именно показанная перестановка каталогов повторно не запускалась. |

Ожидание описывает нужный результат на подготовленных данных, а не копирует
ответ сервера. Ошибочные ответы не использовались как эталон для следующих сборок.
Для полученных через `tools.xquery()` коллекций выполнена попытка полного чтения;
ошибки этого этапа учтены. Результат API
сверен с журналом `xquery_capabilities_actual`. Проверяются и значения, и наличие
полей в каждой строке. Отсутствующее поле не считается полем со значением `null`.
SQL инспектора служит объяснением расхождений, но не заменяет проверку XQuery.

На старых сборках некоторые ошибки SQL дают пустую коллекцию без исключения
в вызывающем коде. На 1525 ошибка может возникнуть только при переборе коллекции,
после возврата из `tools.xquery()`. Поэтому переход **<abbr title="Результат отличается от ожидаемого">≠</abbr> → <abbr title="Ошибка выполнения запроса">О</abbr>** не обязательно
означает новую поломку конструкции: ошибка могла существовать и раньше.

Примеры раскрываются под таблицами. ID, даты, коды и пользовательские поля
относятся к тестовой базе; для своей базы их нужно заменить. Количество записей
в примере — ожидаемое на этих данных, а не постоянное свойство функции.
В ожидаемых JSON-фрагментах показаны только проверяемые поля; платформа может
возвращать дополнительные поля каталога. ID сохранены строками без потери точности.

## Что изменилось между проверенными сборками

| Переход | Наблюдение |
| --- | --- |
| 434 → 906 | Подтвердился явный `join`; внешние JOIN с подходящим порядком каталогов возвращают нужную сторону. На MSSQL прошла форма `return distinct $elem/Fields(...)` без скобок вокруг вызова. |
| 906 → 1132 → 1333 | Статусы всех 69 запросов совпали на соответствующей СУБД. Это не доказывает отсутствие других изменений платформы. |
| 1333 → 1525 | Подтвердились псевдонимы полей, `group by` с `count`, именованный результат `lower`/`upper`, сравнение с `undefined`, `MatchNone` по строковому полю. Агрегаты без группировки прошли на MSSQL. |
| 1333 → 1525 | Изменился порядок каталогов в SQL; текущие внешние JOIN стали сохранять другую сторону. Перестали совпадать результаты `CatalogHierSubset` и проверки экранированного апострофа. |

Фраза «работает на 1525» здесь не означает «появилось ровно в 1525» и не обещает
поддержку на более новых сборках. Промежуточные сборки не проверялись.

## Выборка, поля и простые условия

Имя каталога само по себе выбирает его записи. В `for` можно вернуть всю
запись, перечислить поля через запятую или вызвать `Fields()` со строковыми
именами. Проверена именно форма `Fields('id', 'code', 'name')`; вариант без
кавычек в этот набор не входит.

Сравнения ID `=`, `!=`, `>`, `>=`, `<`, `<=`, строкового кода и логического поля
с `true()` прошли на всех стендах. Равенство пустой ссылке проверено через
`null()`, пустой строке — через `''`, незаполненной дате — через `IsEmpty()`.
Эти проверки не объявляются взаимозаменяемыми для любого типа поля.

**Псевдоним поля — отдельная возможность.** В `return $e/id person_id` новое имя
записывается после выражения, без `as`. На 434–1333 поле `person_id` не появилось
в конечном результате. На 1525 обе СУБД вернули его с правильным ID.
Успешный SQL с псевдонимом на старой сборке ещё не доказывает доступность поля
в объекте, полученном прикладным кодом.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Запрос каталога appointment_types без for | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Поля через запятую | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Fields() со строковыми именами полей | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Возврат всей записи | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Числовой параметр | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение ID типа назначения: != | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение ID типа назначения: > | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение ID типа назначения: >= | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение ID типа назначения: < | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение ID типа назначения: <= | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Строковый параметр | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Равенство true() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение с null() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение с undefined = | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сравнение с пустой строкой | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Проверка IsEmpty() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Псевдоним обычного поля | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "Запрос каталога appointment_types без for"

    ```xquery
    appointment_types
    ```

    Четыре типа назначения: `main` — «Основная», `part` — «Совместитель», `staff` — «Штат», `terminal_contract` — «Срочный договор»; проверяются `id`, `code`, `name`.

??? example "Поля через запятую"

    ```xquery
    for $elem in appointment_types
    return $elem/id, $elem/code, $elem/name
    ```

    Те же четыре типа назначения с правильными `id`, `code`, `name`.

??? example "Fields() со строковыми именами полей"

    ```xquery
    for $elem in appointment_types
    return $elem/Fields('id', 'code', 'name')
    ```

    Те же четыре типа назначения с правильными `id`, `code`, `name`.

??? example "Возврат всей записи"

    ```xquery
    for $elem in collaborators
    where $elem/id = 1105387902724063510
    return $elem
    ```

    Один сотрудник с указанным ID. Проверяются девять полей каталога, включая `fullname`, `code`, `is_dismiss`, даты и сведения о должности; это не проверка полного XML карточки.

??? example "Числовой параметр"

    ```xquery
    for $elem in collaborators
    where $elem/id = 7060866828654770030
    return $elem/Fields('id', 'fullname')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "7060866828654770030",
        "fullname": "Львов Илья Никол[аевич"
      }
    ]
    ```

??? example "Сравнение ID типа назначения: !="

    ```xquery
    for $elem in appointment_types
    where $elem/id != 6820005324597779242
    return $elem/Fields('id', 'name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779239",
        "name": "Основная"
      },
      {
        "id": "6820005324597779240",
        "name": "Совместитель"
      },
      {
        "id": "6820005324597779241",
        "name": "Штат"
      }
    ]
    ```

??? example "Сравнение ID типа назначения: >"

    ```xquery
    for $elem in appointment_types
    where $elem/id > 6820005324597779240
    return $elem/Fields('id', 'name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779241",
        "name": "Штат"
      },
      {
        "id": "6820005324597779242",
        "name": "Срочный договор"
      }
    ]
    ```

??? example "Сравнение ID типа назначения: >="

    ```xquery
    for $elem in appointment_types
    where $elem/id >= 6820005324597779240
    return $elem/Fields('id', 'name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779240",
        "name": "Совместитель"
      },
      {
        "id": "6820005324597779241",
        "name": "Штат"
      },
      {
        "id": "6820005324597779242",
        "name": "Срочный договор"
      }
    ]
    ```

??? example "Сравнение ID типа назначения: <"

    ```xquery
    for $elem in appointment_types
    where $elem/id < 6820005324597779241
    return $elem/Fields('id', 'name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779239",
        "name": "Основная"
      },
      {
        "id": "6820005324597779240",
        "name": "Совместитель"
      }
    ]
    ```

??? example "Сравнение ID типа назначения: <="

    ```xquery
    for $elem in appointment_types
    where $elem/id <= 6820005324597779241
    return $elem/Fields('id', 'name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779239",
        "name": "Основная"
      },
      {
        "id": "6820005324597779240",
        "name": "Совместитель"
      },
      {
        "id": "6820005324597779241",
        "name": "Штат"
      }
    ]
    ```

??? example "Строковый параметр"

    ```xquery
    for $elem in collaborators
    where $elem/code = '13744'
    return $elem/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "1105387902724063510",
        "code": "13744"
      }
    ]
    ```

??? example "Равенство true()"

    ```xquery
    for $elem in collaborators
    where $elem/is_dismiss = true()
    return $elem/Fields('id', 'is_dismiss')
    ```

    Три уволенных сотрудника; у каждой строки `is_dismiss=true`.

??? example "Сравнение с null()"

    ```xquery
    for $elem in collaborators
    where $elem/position_id = null()
    return $elem/Fields('id', 'position_id')
    ```

    Два сотрудника с `position_id=null`.

??? example "Сравнение с undefined ="

    ```xquery
    for $e in collaborators
    where $e/position_id = undefined
    order by $e/id
    return $e/id
    ```

    Два сотрудника без должности, как при сравнении с `null()`; ID отсортированы по возрастанию.

??? example "Сравнение с пустой строкой"

    ```xquery
    for $elem in subdivisions
    where $elem/code = ''
    return $elem/Fields('id', 'code')
    ```

    Четыре подразделения с `code=""`.

??? example "Проверка IsEmpty()"

    ```xquery
    for $elem in collaborators
    where IsEmpty($elem/birth_date) = true()
    return $elem/Fields('id', 'birth_date')
    ```

    Три сотрудника с незаполненным `birth_date`.

??? example "Псевдоним обычного поля"

    ```xquery
    for $e in collaborators
    where $e/id = 1105387902724063510
    return $e/id person_id
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "person_id": "1105387902724063510"
      }
    ]
    ```


## Логические условия

Простые `and` и `or` дали ожидаемые пересечение и объединение условий.
`not($e/code = 'main')` не прошёл ни на одном стенде: SQL содержит форму
`not(code,=,@p0)`. На 434–1333 фактический ответ пустой, на 1525 фиксируется
ошибка выполнения. Поддержку этой формы нельзя выводить из поддержки `and`/`or`.

Для простого сравнения можно использовать отдельно проверенное неравенство
`!=`; это не подтверждение универсального способа заменить любое отрицание.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| not: отрицание условия | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| and: пересечение двух условий | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| or: любое из двух условий | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "not: отрицание условия"

    ```xquery
    for $e in appointment_types
    where not($e/code = 'main')
    return $e/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779240",
        "code": "part"
      },
      {
        "id": "6820005324597779241",
        "code": "staff"
      },
      {
        "id": "6820005324597779242",
        "code": "terminal_contract"
      }
    ]
    ```

??? example "and: пересечение двух условий"

    ```xquery
    for $e in appointment_types
    where $e/id >= 6820005324597779240 and $e/id <= 6820005324597779241
    return $e/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779240",
        "code": "part"
      },
      {
        "id": "6820005324597779241",
        "code": "staff"
      }
    ]
    ```

??? example "or: любое из двух условий"

    ```xquery
    for $e in appointment_types
    where $e/code = 'main' or $e/code = 'staff'
    return $e/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779239",
        "code": "main"
      },
      {
        "id": "6820005324597779241",
        "code": "staff"
      }
    ]
    ```


## Даты

Проверены `date()` без аргумента и преобразование дат вида `YYYY-MM-DD`,
включающая нижняя граница `>=` и исключающая верхняя `<`.
Запрос к `positions/position_finish_date` возвращает три завершённые должности.
Поле находится в каталоге должностей; чтение XML через `__data` здесь не требуется.

Результаты относятся к дате проверки. Выборки с `date()` меняются со временем.
Даты рождения в конечном JSON представлены как `YYYY-MM-DD`, даты завершения
должностей — с временем и смещением. Примеры не устанавливают универсальный
формат сериализации всех полей дат или поведение часовых поясов.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Дата завершения должности раньше текущей даты | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Дата рождения: самый молодой сотрудник | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Дата рождения: самый старший сотрудник | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "Дата завершения должности раньше текущей даты"

    ```xquery
    for $elem in positions
    where $elem/position_finish_date < date()
    return $elem/Fields('id', 'name', 'position_finish_date')
    ```

    Три должности с правильными ID, названиями и датами завершения: `2026-09-14`, `2024-01-03`, `2022-04-19`. В JSON поле `position_finish_date` содержит также время и смещение.

??? example "Дата рождения: самый молодой сотрудник"

    ```xquery
    for $e in collaborators
    where $e/birth_date >= date('2020-08-11')
    return $e/Fields('id', 'fullname', 'birth_date')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "7060856563558727830",
        "fullname": "Белов Дмитрий Анатольевич",
        "birth_date": "2020-08-11"
      }
    ]
    ```

??? example "Дата рождения: самый старший сотрудник"

    ```xquery
    for $e in collaborators
    where $e/birth_date < date('1955-06-11')
    return $e/Fields('id', 'fullname', 'birth_date')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "1105387902724063527",
        "fullname": "Максимова Светлана Викторовна",
        "birth_date": "1955-06-10"
      }
    ]
    ```


## Строковые функции

Двухаргументный `contains()` подтвердился на обеих СУБД во всех сборках.
Дополнительный аргумент `true()` проверялся как ограничение началом строки:
для шаблона `t` ожидается только `terminal_contract`. MSSQL возвращает эту запись;
PostgreSQL на всех пяти сборках ищет `%t%` и возвращает также `part` и `staff`.
На PostgreSQL эту форму нельзя использовать как подтверждённый поиск префикса.

`substring(code, 4, 1)` прошёл на MSSQL. На PostgreSQL позиции передаются
как `bigint`, и SQL сообщает об отсутствии функции
`substring(character varying, bigint, bigint)`. На старых сборках это выражается
пустой коллекцией, на 1525 — исключением при её чтении.

Для `lower()` и `upper()` проверены **значения в именованном поле результата**,
а не только выполнение функции в SQL. Такое поле появляется на обеих СУБД
в проверке 1525; на 434–1333 конечный результат ожиданию не соответствует.
Регистр, сортировки и поиск по другим алфавитам отдельно не исследовались.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Поиск подстроки через contains() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| substring: символ 4 равен 3 | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| lower: точные значения в результате | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| upper: точные значения в результате | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| contains: поиск по началу строки | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> |

??? example "Поиск подстроки через contains()"

    ```xquery
    for $elem in appointment_types
    where contains($elem/name, 'Основная')
    return $elem/Fields('id', 'name')
    ```

    Один тип назначения: «Основная».

??? example "substring: символ 4 равен 3"

    ```xquery
    for $e in collaborators
    where substring($e/code, 4, 1) = '3'
    return $e/id
    ```

    Один сотрудник с ID `7060869043305472936`: четвёртый символ его кода равен `3`.

??? example "lower: точные значения в результате"

    ```xquery
    for $e in appointment_types
    return $e/id, lower($e/name) normalized_name
    ```

    Четыре ID и соответствующие значения `normalized_name`: `основная`, `совместитель`, `штат`, `срочный договор`.

??? example "upper: точные значения в результате"

    ```xquery
    for $e in appointment_types
    return $e/id, upper($e/name) normalized_name
    ```

    Четыре ID и соответствующие значения `normalized_name`: `ОСНОВНАЯ`, `СОВМЕСТИТЕЛЬ`, `ШТАТ`, `СРОЧНЫЙ ДОГОВОР`.

??? example "contains: поиск по началу строки"

    ```xquery
    for $e in appointment_types
    where contains($e/code, 't', true())
    return $e/Fields('id', 'code')
    ```

    Один тип назначения с кодом `terminal_contract`; `part` и `staff` не должны попасть в результат.


## Литералы и арифметика

Отрицательное и шестнадцатеричное целое, дробное число `5.5`, операции
`+`, `-`, `*`, `/` и `div` прошли во всех пяти сборках на обеих СУБД.
В примерах арифметика выбирает должность со ставкой `basic_rate=11`.
Проверка `22 div 2` не устанавливает правила округления или поведения дробного
частного. Переполнение и деление на ноль этим набором не проверялись.

`23 mod 12` не прошёл: в SQL остаётся `@p0mod@p1` вместо операции остатка.
На старых сборках получена пустая коллекция, на 1525 — ошибка выполнения.

Сравнение `position_id = undefined` прошло только на 1525. На 434–1333
`undefined` попадает в SQL как имя колонки. Для пустой ссылки во всех пяти
сборках отдельно подтвердилось `position_id = null()`.
Результат не распространяется автоматически на форму `undefined()`.

**Экранирование апострофа изменилось.** Условие `'a''b' = "a'b"` на 434–1333
даёт четыре записи каталога типов назначения. На 1525 обе СУБД возвращают
пустую коллекцию: в параметры передаются разные строки — `a''` и `a'b`.
Это наблюдение о разборе конкретного литерала, а не о любых кавычках в XQuery.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Ставка 11: 5.5 * 2 = basic_rate | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Удвоенная одинарная кавычка | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> |
| Отрицательное целое | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Шестнадцатеричное целое | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: 5 + 6 = basic_rate | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: 20 - 9 = basic_rate | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: basic_rate * 2 = 22 | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: 22 / 2 = basic_rate | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: 22 div 2 = basic_rate | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Ставка 11: 23 mod 12 = basic_rate | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |

??? example "Ставка 11: 5.5 * 2 = basic_rate"

    ```xquery
    for $e in positions
    where (5.5 * 2 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Удвоенная одинарная кавычка"

    ```xquery
    for $elem in appointment_types
    where 'a''b' = "a'b"
    return $elem/Fields('id')
    ```

    Все четыре ID типов назначения: сравниваемые строки должны быть равны.

??? example "Отрицательное целое"

    ```xquery
    for $elem in appointment_types
    where -1 < 0
    return $elem/Fields('id')
    ```

    Все четыре ID типов назначения: условие `-1 < 0` истинно.

??? example "Шестнадцатеричное целое"

    ```xquery
    for $elem in collaborators
    where $elem/id = 0x61FD3CF8784E776E
    return $elem/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "7060866828654770030",
        "code": ""
      }
    ]
    ```

??? example "Ставка 11: 5 + 6 = basic_rate"

    ```xquery
    for $e in positions
    where (5 + 6 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Ставка 11: 20 - 9 = basic_rate"

    ```xquery
    for $e in positions
    where (20 - 9 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Ставка 11: basic_rate * 2 = 22"

    ```xquery
    for $e in positions
    where ($e/basic_rate * 2 = 22)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Ставка 11: 22 / 2 = basic_rate"

    ```xquery
    for $e in positions
    where (22 / 2 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Ставка 11: 22 div 2 = basic_rate"

    ```xquery
    for $e in positions
    where (22 div 2 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.

??? example "Ставка 11: 23 mod 12 = basic_rate"

    ```xquery
    for $e in positions
    where (23 mod 12 = $e/basic_rate)
    return $e/id
    ```

    Один ID `7060452475378420334` — должность со ставкой `basic_rate=11`.


## MatchSome и MatchNone

`MatchSome()` прошёл и для строкового кода, и для многозначного `category_id`.
У сотрудника из примера категория `["123"]`; список `('123', '456')` находит его.

`MatchNone()` нужно рассматривать отдельно для каждого вида аргументов:

- Строковое поле и список исключений: на 1525 обе СУБД возвращают три типа
  назначения без `main`; на 434–1333 правильный результат не получен.
- Многозначное поле: на 1525 PostgreSQL запись с `["123"]` остаётся при списке
  `('456', '789')`. На MSSQL результат пустой: SQL содержит положительную
  проверку `.exist(...)=1`, без нужного отрицания.
- Подзапрос: ни один стенд не дал ожидаемого результата. На 1525 SQL смешивает
  псевдонимы внешней и внутренней выборок, и обе СУБД возвращают ошибку.

Дополнительно на **1525 PostgreSQL** проверены два списка для `["123"]`:
`('123', '456')` исключает запись, `('456', '789')` оставляет её.
Для этих данных достаточно одного пересечения для исключения;
SQL имеет вид `not(category_id && @p1) or category_id is null`.
`MatchSome(...) = false()` дал те же результаты в этих двух опытах,
а `not(MatchSome(...))` сформировал ошибочный SQL. Эти дополнительные наблюдения
не распространяются на MSSQL или другие сборки. Поведение всех вариантов
пустых списков и отсутствующих значений отдельно не установлено.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| MatchSome() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| MatchSome() для множественного поля | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| MatchNone по строковому коду | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| MatchNone с вложенной выборкой | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| MatchNone: множественное поле и список исключений | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "MatchSome()"

    ```xquery
    for $elem in appointment_types
    where MatchSome($elem/code, ('main', 'part'))
    return $elem/Fields('id', 'code')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "6820005324597779239",
        "code": "main"
      },
      {
        "id": "6820005324597779240",
        "code": "part"
      }
    ]
    ```

??? example "MatchSome() для множественного поля"

    ```xquery
    for $elem in collaborators
    where MatchSome($elem/category_id, ('123', '456'))
    return $elem/Fields('id', 'fullname', 'category_id')
    ```

    Один сотрудник с ID `7060870052597727632`, правильным ФИО и `category_id=["123"]`.

??? example "MatchNone по строковому коду"

    ```xquery
    for $e in appointment_types
    where MatchNone($e/code, ('main'))
    order by $e/id
    return $e/id
    ```

    Три ID типов назначения без `main`, в порядке возрастания: `part`, `staff`, `terminal_contract`.

??? example "MatchNone с вложенной выборкой"

    ```xquery
    for $e in appointment_types
    where MatchNone($e/id, (
            for $x in appointment_types
            where $x/id = 6820005324597779239
            return $x/id
        ))
    order by $e/id
    return $e/id
    ```

    Те же три ID: подзапрос должен исключить только тип «Основная».

??? example "MatchNone: множественное поле и список исключений"

    ```xquery
    for $e in collaborators
    where $e/id = 7060870052597727632 and MatchNone($e/category_id, ('456', '789'))
    return $e/Fields('id', 'category_id')
    ```

    Один объект с ID `7060870052597727632` и `category_id=["123"]`: пересечения со списком исключений нет.


## Уникальные значения, группировка и агрегаты

Две формы уникальной выборки проверялись раздельно:
`return distinct($elem/Fields(...))` и `return distinct $elem/Fields(...)`.
У трёх уволенных сотрудников одно подразделение; ожидается массив
с одним объектом `{"position_parent_name":"PR-отдел"}`.

На MSSQL форма со скобками прошла во всех пяти сборках, форма без скобок —
на 906, 1132, 1333 и 1525. На PostgreSQL обе формы не прошли: автоматически
добавленный `order by id` конфликтует с выборкой `DISTINCT`, в которую `id`
не входит. Сам по себе корректный SQL `SELECT DISTINCT` без этого дополнения
не подтверждает работу исходного XQuery.

`group by is_dismiss` с `count(id) total` на 1525 возвращает две группы:
54 действующих и 3 уволенных сотрудника. На 434–1333 этот запрос не прошёл.
Это проверка конкретной группировки с `count`, а не всех сочетаний агрегатов.

Для агрегатов без группировки имя результата также существенно:
`return sum($e/basic_rate) value` должно дать объект с полем `value`.
На MSSQL 1525 подтвердились все пять примеров `count`, `min`, `max`, `sum`, `avg`.
На старых MSSQL SQL мог вычислить агрегат без ошибки, но нужного поля в конечном
объекте не было. На PostgreSQL 1525 все пять запросов завершаются ошибкой:
автоматический `order by id` несовместим с агрегатной выборкой без группировки.

В каталоге 57 должностей; ставка заполнена только у трёх: **5, 5 и 11**.
Ожидаемая сумма — **21**, среднее по заполненным ставкам — **7**.
Незаполненные ставки не считаются нулями. Проверка целого среднего не описывает
точность и округление дробных результатов. Значения агрегатов в проверенном
конечном JSON приходят строками, например `{"value":"7"}`.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| distinct() от Fields(): подразделения уволенных | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| Модификатор distinct перед Fields() | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| group by: count | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| count с псевдонимом | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| min с псевдонимом | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| max с псевдонимом | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| Сумма ставок должностей | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |
| Средняя ставка должностей | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |

??? example "distinct() от Fields(): подразделения уволенных"

    ```xquery
    for $elem in collaborators
    where $elem/is_dismiss = true()
    return distinct($elem/Fields('position_parent_name'))
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "position_parent_name": "PR-отдел"
      }
    ]
    ```

??? example "Модификатор distinct перед Fields()"

    ```xquery
    for $elem in collaborators
    where $elem/is_dismiss = true()
    return distinct $elem/Fields('position_parent_name')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "position_parent_name": "PR-отдел"
      }
    ]
    ```

??? example "group by: count"

    ```xquery
    for $e in collaborators
    group by $e/is_dismiss
    order by $e/is_dismiss
    return $e/is_dismiss, count($e/id) total
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "is_dismiss": false,
        "total": "54"
      },
      {
        "is_dismiss": true,
        "total": "3"
      }
    ]
    ```

??? example "count с псевдонимом"

    ```xquery
    for $e in collaborators
    return count($e/id) value
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "value": "57"
      }
    ]
    ```

??? example "min с псевдонимом"

    ```xquery
    for $e in collaborators
    return min($e/id) value
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "value": "1105387902724063494"
      }
    ]
    ```

??? example "max с псевдонимом"

    ```xquery
    for $e in collaborators
    return max($e/id) value
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "value": "7060872677648127560"
      }
    ]
    ```

??? example "Сумма ставок должностей"

    ```xquery
    for $e in positions
    return sum($e/basic_rate) value
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "value": "21"
      }
    ]
    ```

??? example "Средняя ставка должностей"

    ```xquery
    for $e in positions
    return avg($e/basic_rate) value
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "value": "7"
      }
    ]
    ```


## Связи каталогов и направление JOIN

На эталоне четыре типа назначения и 57 должностей. Только две должности связаны
с типами назначения: одна с «Основная», другая с «Совместитель». Для внутреннего
соединения ожидаются две строки; для сохранения всех типов назначения — четыре,
включая «Штат» и «Срочный договор» с `basic_collaborator_id=null`.

Перечень каталогов через запятую и `some ... satisfies` прошли на всех стендах.
Явный `join` дал ожидаемый результат на 906, 1132, 1333 и 1525; на 434 — ошибка.
`ForeignElem(position_id)/name` также прошёл везде. Путь
`position_id/ForeignDispName` в условии не прошёл ни на одном стенде:
провайдер пытается обратиться к SQL-колонке `ForeignDispName`.

### Как меняется порядок каталогов

Здесь `A` — каталог после `for`, `B` — второй каталог. Таблица описывает
**порядок в сформированном SQL**, а не порядок чтения таблиц оптимизатором СУБД.

| Форма XQuery | 434 | 906, 1132, 1333 | 1525 |
| --- | --- | --- | --- |
| `for A, B` | `FROM B, A` | `FROM B, A` | `FROM A, B` |
| `for A join B` | Ошибка | `B INNER JOIN A` | `A INNER JOIN B` |
| `for A ljoin B` | <abbr title="JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> | `B LEFT JOIN A` — сохраняется B | `A LEFT JOIN B` — сохраняется A |
| `for A rjoin B` | <abbr title="JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> | `B RIGHT JOIN A` — сохраняется A | `A RIGHT JOIN B` — сохраняется B |

Порядок подтверждён на обеих СУБД для приведённых соединений двух каталогов.
Для внутренних соединений с проверенным равенством две нужные строки сохранились
при перестановке. Для внешних соединений сторона сохранения изменилась.

В общем наборе `ljoin` начинается с `positions`, а `rjoin` — с
`appointment_types`: это порядок, который сохраняет типы назначения на 906–1333.
На 1525 те же запросы сохраняют **57 должностей**, поэтому отмечены **<abbr title="Результат отличается от ожидаемого">≠</abbr>**.
Это не означает, что `ljoin` и `rjoin` в целом не работают на 1525.

Перестановка каталогов дополнительно проверена на 1525 MSSQL и PostgreSQL.
Оба изменённых запроса возвращают четыре типа назначения. Однако только `rjoin`
полностью совпал с ожиданием: `ljoin` возвращает для двух типов без должностей
`basic_collaborator_id=""` вместо `null`. Запросы и результаты этой дополнительной
проверки приведены ниже. Статусы основного набора относятся к исходным запросам
и сохраняются, чтобы изменение поведения оставалось видимым.

Для 434 внешние соединения помечены **<abbr title="JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr>** по предыдущим ошибкам JOIN
и принятому решению не повторять тест нового порядка каталогов. В этих четырёх
ячейках нет результата повторного исполнения показанных запросов.
Цепочки из трёх и более каталогов не проверялись.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Соединение перечнем каталогов: типы назначения и должности | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| some/satisfies: типы назначения с должностями | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| join: типы назначения и должности | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| ljoin: типы назначения и должности | <abbr title="MSSQL: JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> / <abbr title="PostgreSQL: JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> |
| rjoin: типы назначения и должности | <abbr title="MSSQL: JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> / <abbr title="PostgreSQL: JOIN на 434 не прошёл проверку; новый порядок каталогов не перепроверялся">Н&#42;</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> |
| ForeignElem() по позиции Архитектор | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| ForeignDispName в условии | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> | <abbr title="MSSQL: Ошибка выполнения запроса">О</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |

??? example "Соединение перечнем каталогов: типы назначения и должности"

    ```xquery
    for $elem in appointment_types, $pos in positions
    where $elem/id = $pos/position_appointment_type_id
    return $elem/id, $elem/name, $pos/basic_collaborator_id
    ```

    Те же две связи типов назначения с должностями; дополнительно проверяется название типа назначения.

??? example "some/satisfies: типы назначения с должностями"

    ```xquery
    for $e in appointment_types
    where some $p in positions satisfies ($e/id = $p/position_appointment_type_id)
    return $e/Fields('id', 'name')
    ```

    Два типа назначения с должностями: «Основная» и «Совместитель»; поля `id`, `name`.

??? example "join: типы назначения и должности"

    ```xquery
    for $e in appointment_types
    join $p in positions
    on $e/id = $p/position_appointment_type_id
    return $e/id, $p/basic_collaborator_id
    ```

    Две строки: «Основная» → сотрудник `1105387902724063510`, «Совместитель» → `7060870052597727632`. Проверяются ID типа назначения и сотрудника.

??? example "ljoin: типы назначения и должности"

    ```xquery
    for $p in positions
    ljoin $e in appointment_types
    on $e/id = $p/position_appointment_type_id
    return $e/id, $p/basic_collaborator_id
    ```

    Четыре типа назначения: для «Основная» и «Совместитель» — ID назначенных сотрудников, для «Штат» и «Срочный договор» — `basic_collaborator_id=null`.

??? example "rjoin: типы назначения и должности"

    ```xquery
    for $e in appointment_types
    rjoin $p in positions
    on $e/id = $p/position_appointment_type_id
    return $e/id, $p/basic_collaborator_id
    ```

    Те же четыре типа назначения и значения `basic_collaborator_id`, что в примере `ljoin`.

??? example "ForeignElem() по позиции Архитектор"

    ```xquery
    for $elem in collaborators
    where ForeignElem($elem/position_id)/name = 'Архитектор'
    return $elem/Fields('id', 'position_id')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "7060870052597727632",
        "position_id": "7060454080989654281"
      }
    ]
    ```

??? example "ForeignDispName в условии"

    ```xquery
    for $elem in collaborators
    where $elem/position_id/ForeignDispName = 'Архитектор'
    return $elem/Fields('id', 'position_id')
    ```

    Ожидаемый результат по проверяемым полям:

    ```json
    [
      {
        "id": "7060870052597727632",
        "position_id": "7060454080989654281"
      }
    ]
    ```


### Внешние соединения на 1525 после перестановки

Это два дополнительных варианта к общему набору. На обеих СУБД проверены
четыре строки, поля `id`, `basic_collaborator_id` и их значения; результаты
API совпали с фактическими журналами. Ожидания остались прежними:
два связанных сотрудника и два `null` для типов назначения без должностей.

| Вариант для 1525 | MSSQL | PostgreSQL |
| --- | --- | --- |
| `appointment_types ljoin positions` | <abbr title="MSSQL: Результат отличается от ожидаемого — пустая строка вместо null">≠</abbr> | <abbr title="PostgreSQL: Результат отличается от ожидаемого — пустая строка вместо null">≠</abbr> |
| `positions rjoin appointment_types` | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

**`rjoin`: подтверждённый вариант для получения четырёх типов назначения на 1525.**

```xquery
for $p in positions
rjoin $e in appointment_types
on $e/id = $p/position_appointment_type_id
return $e/id, $p/basic_collaborator_id
```

SQL содержит `positions RIGHT JOIN appointment_types`. Конечный результат
по проверяемым полям, без требования к порядку строк:

```json
[
  {
    "id": "6820005324597779239",
    "basic_collaborator_id": "1105387902724063510"
  },
  {
    "id": "6820005324597779240",
    "basic_collaborator_id": "7060870052597727632"
  },
  {
    "id": "6820005324597779241",
    "basic_collaborator_id": null
  },
  {
    "id": "6820005324597779242",
    "basic_collaborator_id": null
  }
]
```

**`ljoin`: нужные четыре записи получены, но значения пустой связи отличаются.**

```xquery
for $e in appointment_types
ljoin $p in positions
on $e/id = $p/position_appointment_type_id
return $e/id, $p/basic_collaborator_id
```

SQL содержит `appointment_types LEFT JOIN positions` и выполняется без ошибки.
Первые две записи совпадают с ожиданием. В последних двух конечный результат
содержит пустые строки:

```json
[
  {"id": "6820005324597779241", "basic_collaborator_id": ""},
  {"id": "6820005324597779242", "basic_collaborator_id": ""}
]
```

Поля присутствуют, но `""` и `null` — разные значения. Поэтому этот вариант
не отмечен как полностью пройденный. Равное количество строк и успешный SQL
не делают результаты двух внешних соединений эквивалентными для прикладного кода.
Наблюдение относится к указанной проекции и данным, а не ко всем полям любых JOIN.

## Иерархия

`IsHierChild()` вернул пять потомков коммерческого департамента,
`IsHierChildOrSelf()` — те же подразделения и сам департамент. Обе формы прошли
на всех стендах. В примерах присутствует `order by $elem/Hier()`, но проверялся
состав строк без требования к их порядку. Поэтому отдельного подтверждения
точного порядка обхода иерархии здесь нет.

`CatalogHierSubset()` на 434–1333 вернул ожидаемые `id` и `name` пяти потомков.
На 1525 SQL выполняется, но конечный JSON выглядит как `[{}, {}, {}, {}, {}]`.
Диагностика объектов перед сериализацией также не обнаружила ожидаемых полей.
Отдельная ручная проверка подтвердила потерю полей и возврат пустых объектов.
Подтверждено нарушение этого сценария получения результата; из него нельзя
заключать, что рекурсивный SQL вообще не находит нужные подразделения.
Для выборки потомков с явными полями проверен пример с `IsHierChild()`.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| IsHierChild() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| IsHierChildOrSelf() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| CatalogHierSubset() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат отличается от ожидаемого">≠</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> |

??? example "IsHierChild()"

    ```xquery
    for $elem in subdivisions
    where IsHierChild($elem/id, 7059746249784127958)
    order by $elem/Hier()
    return $elem/Fields('id', 'name')
    ```

    Пять потомков: PR-отдел, отдел маркетинга, отдел продаж и две подчинённые группы. Проверяются `id` и `name`, без требования к порядку.

??? example "IsHierChildOrSelf()"

    ```xquery
    for $elem in subdivisions
    where IsHierChildOrSelf($elem/id, 7059746249784127958)
    order by $elem/Hier()
    return $elem/Fields('id', 'name')
    ```

    Шесть подразделений: те же пять потомков и коммерческий департамент. Проверяются `id` и `name`, без требования к порядку.

??? example "CatalogHierSubset()"

    ```xquery
    CatalogHierSubset('subdivisions', 7059746249784127958)
    ```

    Пять потомков с полями `id`, `name`, как в примере `IsHierChild()`.


## Поиск по документу и пользовательским полям

Все четыре приведённые формы `doc-contains()` прошли на обеих СУБД во всех
пяти сборках. Обычный текстовый поиск и выражения пользовательских полей
обращаются к разным данным, хотя записываются одной функцией.

Для поиска текста `Анисимов` на стендах включён `LuceneFTIndex=True`.
Ожидаются **четыре сотрудника**, поскольку слово встречается не только в ФИО,
но и в других данных документа, например в сведениях о руководителе.
Такой поиск не равнозначен `contains(fullname, 'Анисимов')`.

Условия в квадратных скобках в этих прогонах переводятся в SQL-проверки
пользовательских полей XML. `f_zsbu = зеленый` находит одну запись,
`f_zsbu contains зелен` — две, а `is_universal = true~bool` — одну.
Суффикс `~bool` является частью проверенного выражения для логического значения.
Эти три проверки не требовали подтверждения полнотекстового индекса в наборе;
отдельный прогон с отключённым `LuceneFTIndex` не выполнялся.

Из этих примеров не следует поддержка всех возможностей поискового языка:
точные фразы, кавычки, специальные символы, свежесть индекса и сложные комбинации
условий должны проверяться отдельно. Наличие XML-документа без соответствующей
записи в `collaborators` не добавляет его в выборку из этого каталога.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Полнотекстовый doc-contains() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Равенство кастомного поля в doc-contains() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Подстрока кастомного поля в doc-contains() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Булево кастомное поле в doc-contains() | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "Полнотекстовый doc-contains()"

    ```xquery
    for $elem in collaborators
    where doc-contains($elem/id, '', 'Анисимов')
    return $elem/Fields('id', 'fullname')
    ```

    Четыре записи: Тихомирова, Анисимов, Вилкова и Лезова. Проверяются соответствующие ID и полные ФИО.

??? example "Равенство кастомного поля в doc-contains()"

    ```xquery
    for $elem in collaborators
    where doc-contains($elem/id, '', '[f_zsbu = зеленый]')
    return $elem/Fields('id', 'fullname')
    ```

    Один сотрудник, у которого `f_zsbu` содержит ровно `зеленый`.

??? example "Подстрока кастомного поля в doc-contains()"

    ```xquery
    for $elem in collaborators
    where doc-contains($elem/id, '', '[f_zsbu contains зелен]')
    return $elem/Fields('id', 'fullname')
    ```

    Два сотрудника: значения `f_zsbu` — `зеленый` и `зелен`.

??? example "Булево кастомное поле в doc-contains()"

    ```xquery
    for $elem in collaborators
    where doc-contains($elem/id, '', '[is_universal = true~bool]')
    return $elem/Fields('id', 'fullname')
    ```

    Один сотрудник с `is_universal=true`.


## Сортировка

Проверены сортировка названия по возрастанию, `descending` и два ключа
`is_dynamic ascending, name descending`. Все три запроса прошли на всех стендах.
Для одного ключа проверен точный порядок ожидаемых строк; для двух — порядок
по указанным ключам и состав данных. Порядок при равных ключах не фиксировался.

Для запросов без `order by` порядок результата не является частью ожидания.
Кроме того, провайдер может добавлять сортировку самостоятельно: в проверках
PostgreSQL это влияет на `DISTINCT`, агрегаты и объединение каталогов.
Порядок имён на этих данных не устанавливает одинаковое поведение всех
сортировок СУБД для регистра, символов и языков.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Сортировка типов назначения: $elem/name | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сортировка типов назначения: $elem/name descending | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |
| Сортировка групп по типу и названию | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат совпал с ожиданием">✓</abbr> |

??? example "Сортировка типов назначения: $elem/name"

    ```xquery
    for $elem in appointment_types
    order by $elem/name
    return $elem/Fields('id', 'name')
    ```

    Четыре типа в порядке «Основная», «Совместитель», «Срочный договор», «Штат», с соответствующими ID.

??? example "Сортировка типов назначения: $elem/name descending"

    ```xquery
    for $elem in appointment_types
    order by $elem/name descending
    return $elem/Fields('id', 'name')
    ```

    Четыре типа в обратном порядке: «Штат», «Срочный договор», «Совместитель», «Основная».

??? example "Сортировка групп по типу и названию"

    ```xquery
    for $elem in groups
    order by $elem/is_dynamic ascending, $elem/name descending
    return $elem/Fields('id', 'name', 'is_dynamic')
    ```

    Сначала две обычные группы по убыванию названия («Учебный центр», «Учебная группа с открытым вступлением»), затем «Динамическая группа»; проверяются `id`, `name`, `is_dynamic`.


## Объединение каталогов

Выражение `for $elem in (groups, appointment_types)` проверяет объединение
записей двух каталогов, а не соединение строк по условию. Ожидаются семь ID:
три группы и четыре типа назначения. На MSSQL запрос прошёл во всех сборках.

На PostgreSQL 434–1333 SQL обращается к `groups` без схемы `dbo` и на этих
стендах сообщает, что отношение не найдено. Это ограничение наблюдаемого SQL
и окружения, а не доказательство отсутствия самой операции объединения.
На 1525 имена таблиц уже квалифицированы схемой, но добавленный `order by`
ссылается на `t_elem` вне области этого псевдонима и вызывает другую ошибку.

| Проверенная форма | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| Объединение групп и типов назначения | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Результат отличается от ожидаемого">≠</abbr> | <abbr title="MSSQL: Результат совпал с ожиданием">✓</abbr> / <abbr title="PostgreSQL: Ошибка выполнения запроса">О</abbr> |

??? example "Объединение групп и типов назначения"

    ```xquery
    for $elem in (groups, appointment_types)
    return $elem/id
    ```

    Семь ID — три группы и четыре типа назначения. Порядок не фиксируется; количество повторов проверяется.

## Как применять результаты

При выборе конструкции смотрите на сборку, СУБД и конкретную форму выражения.
Если меняется сборка, сначала проверяйте результат запросов, в которых есть
внешние соединения, новые имена полей, агрегаты, `DISTINCT`, `MatchNone`,
иерархическая выборка или экранирование строк. Поддержка соседней формы
или удачный результат на другой СУБД не заменяют такую проверку.

Для своего запроса задайте ожидаемые строки и поля на известных данных,
прочитайте всю коллекцию и проверьте значения. Если нужны уникальные подразделения,
правильный результат — уникальные объекты с нужным полем, а не просто «одна строка».
Если ожидается `person_id`, наличие исходного `id` вместо него — расхождение.
Дополнительные поля каталога при этом сами по себе не считаются ошибкой проекции.

При расхождении смотрите именно **исполненный SQL**, включая автоматически
добавленные `ORDER BY`, ограничения выборки и параметры. Успешный ручной запуск
упрощённого SQL не доказывает корректность исходного XQuery. Подход и инструмент
описаны в разделе [«Преобразование XQuery в SQL»](../xquery-to-sql/index.md).

Набор проверяет основные конструкции на небольших каталогах. Он не устанавливает
поведение больших выборок и постраничного чтения, производительность, все
варианты типов, пустых значений и сочетаний функций. Результаты получены
с отключённым `XQueryTemplateCache`; включённый кэш требует отдельной проверки.
