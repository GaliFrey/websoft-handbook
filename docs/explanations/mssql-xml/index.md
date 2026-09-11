# XML-поля WebSoft HCM в Microsoft SQL Server

Страница показывает, как читать и преобразовывать XML-документы объектов
WebSoft HCM прямыми запросами к Microsoft SQL Server. Примеры проверены на
WebSoft HCM Server `2023.2.906` с MSSQL provider `1.24.4.18` и Microsoft SQL
Server `16.0.4275.2`.

## Каталог и XML-документ

Для сотрудника используются две таблицы:

| Таблица | Назначение |
| --- | --- |
| `dbo.collaborators` | Каталог с основными полями, пригодными для обычной выборки и фильтрации |
| `dbo.collaborator` | Документная таблица с полным XML в колонке `data` |

Если значение уже присутствует в `dbo.collaborators`, следует читать его из
каталога. Обращение к `dbo.collaborator.data` оправдано для данных, которых нет
в каталоге: например, повторяющихся или кастомных полей.

Во всех запросах к документной таблице используется ID тестового сотрудника
`1105387902724063510`.

## Типизированный и нетипизированный XML

Колонка типа `xml` может быть связана с коллекцией XML-схем. Такая колонка
называется типизированной; без коллекции схем XML считается нетипизированным.
Связь `dbo.collaborator.data` с коллекцией можно проверить запросом:

```sql
SELECT
  C.name AS column_name,
  TYPE_NAME(C.user_type_id) AS data_type,
  SCHEMA_NAME(XSC.schema_id) AS collection_schema,
  XSC.name AS collection_name
FROM sys.columns AS C
LEFT JOIN sys.xml_schema_collections AS XSC
  ON XSC.xml_collection_id = C.xml_collection_id
WHERE C.object_id = OBJECT_ID(N'dbo.collaborator')
  AND C.name = N'data';
```

Если `collection_name` равен `NULL`, колонка не связана с коллекцией схем.
Типизированный XML проверяется по схеме, а статическая типизация XQuery может
потребовать более точных путей и явных преобразований.

!!! info "Значения типизированных элементов"
    На проверенном стенде `dbo.collaborator.data` типизирован, а элемент
    `collaborator/id` определён как `xs:long`. Для простых типизированных
    элементов шаг `text()` не поддерживается. Метод `value()` должен выбирать
    сам элемент, например `(/collaborator/id)[1]`. Когда внутри XQuery нужна
    именно строка, используется `string()`.

## Методы типа `xml`

| Метод | Назначение | Результат |
| --- | --- | --- |
| `query()` | Возвращает выбранные узлы или сконструированный XML | Нетипизированный `xml` |
| `value()` | Извлекает одно значение и преобразует его в тип SQL Server | Скалярное значение |
| `exist()` | Проверяет, возвращает ли XQuery непустую последовательность | `bit`: `1`, `0` или `NULL` |
| `nodes()` | Создаёт строку для каждого выбранного XML-узла | Табличный набор |
| `modify()` | Выполняет одно выражение XML DML | Изменяет XML-переменную или колонку |

Методы принимают выражения из поддерживаемого SQL Server подмножества XQuery.
Обычные пути XPath являются частью этого синтаксиса.

## Выбор XML-узлов через `query()`

### Первый и все одноимённые узлы

```sql
SELECT
  C.data.query('(/collaborator/lastname)[1]') AS first_lastname,
  C.data.query(
    '/collaborator/custom_elems/custom_elem/name'
  ) AS custom_field_names
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Выражение в первом столбце возвращает первый `<lastname>`. Во втором нет
позиционного предиката, поэтому `query()` возвращает последовательность всех
найденных `<name>` как одно значение типа `xml`.

### Узел вместе с содержимым

```sql
SELECT C.data.query('/collaborator/custom_elems') AS custom_elems
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

В результат входит `<custom_elems>` вместе со всеми вложенными
`<custom_elem>`.

### Узел по позиции родителя

```sql
SELECT C.data.query(
  '/collaborator/custom_elems/custom_elem[2]/name'
) AS second_custom_field_name
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Предикат `[2]` применяется к `<custom_elem>`, поэтому запрос возвращает
`<name>` из второго кастомного поля.

### Родительский узел по значению дочернего

```sql
SELECT C.data.query(
  '(/collaborator/custom_elems/custom_elem[name = "fld_city"])[1]'
) AS city_custom_elem
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Условие в квадратных скобках отбирает `<custom_elem>`, внутри которого есть
`<name>fld_city</name>`.

### Значение соседнего узла

```sql
SELECT C.data.query(
  '(/collaborator/custom_elems/custom_elem[name = "fld_city"]/value)[1]'
) AS city_value_node
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

После отбора родительского `<custom_elem>` путь продолжается до соседнего
элемента `<value>`.

## Получение значений через `value()`

`value()` должен получить не более одного значения. Даже когда в документе
ожидается единственный элемент, внешнее `[1]` явно сообщает статическому
анализатору XQuery, что выражение возвращает скаляр.

```sql
SELECT
  C.data.value('(/collaborator/id)[1]', 'bigint') AS xml_id,
  C.data.value(
    '(/collaborator/lastname)[1]',
    'nvarchar(100)'
  ) AS lastname,
  C.data.value('(/collaborator/birth_date)[1]', 'date') AS birth_date,
  C.data.value('(/collaborator/is_dismiss)[1]', 'bit') AS is_dismiss
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Неверное преобразование завершается ошибкой SQL Server. Например, фамилия не
может быть преобразована в дату:

```sql
SELECT C.data.value('(/collaborator/lastname)[1]', 'date')
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### Значения кастомных полей

```sql
SELECT
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[name = "fld_city"]/value)[1]',
    'nvarchar(100)'
  ) AS city,
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[name = "fld_birth_date"]/value)[1]',
    'datetimeoffset(0)'
  ) AS custom_birth_date,
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[name = "fld_skills"]/value)[1]',
    'nvarchar(max)'
  ) AS skills
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Для значения с часовым смещением используется `datetimeoffset`, чтобы не
терять часть `+00:00`.

### Первое подходящее состояние

```sql
SELECT
  C.data.value(
    '(/collaborator/history_states/history_state[state_id = "vacation"]/start_date)[1]',
    'datetimeoffset(0)'
  ) AS vacation_start,
  C.data.value(
    '(/collaborator/history_states/history_state[state_id = "vacation"]/finish_date)[1]',
    'datetimeoffset(0)'
  ) AS vacation_finish
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### Последние узлы

```sql
SELECT
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[last()]/name)[1]',
    'nvarchar(100)'
  ) AS last_name,
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[last() - 1]/name)[1]',
    'nvarchar(100)'
  ) AS previous_name,
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[last() - 2]/name)[1]',
    'nvarchar(100)'
  ) AS third_from_end_name
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Если подходящего узла нет, `value()` возвращает `NULL`.

## Проверка узлов через `exist()`

`exist()` возвращает `1` для непустой последовательности, `0` для пустой и
`NULL`, если сам экземпляр `xml` равен `NULL`.

### Отсутствующий, пустой и заполненный узел

```sql
DECLARE @samples TABLE (
  label nvarchar(20) NOT NULL,
  data xml NULL
);

INSERT INTO @samples (label, data)
VALUES
  (N'absent', N'<collaborator/>'),
  (N'empty', N'<collaborator><code/></collaborator>'),
  (N'filled', N'<collaborator><code>demo</code></collaborator>'),
  (N'null', NULL);

SELECT
  S.label,
  S.data.exist('/collaborator/code') AS node_exists,
  S.data.exist('/collaborator/code/text()') AS text_exists
FROM @samples AS S
ORDER BY S.label;
```

Наличие элемента и наличие текстового узла — разные проверки. Пустой
`<code/>` существует, но не содержит `text()`.

!!! warning "`exist()` проверяет непустую последовательность"
    Выражения `data.exist('true()')` и `data.exist('false()')` оба возвращают
    `1`: каждое создаёт непустое скалярное значение. Для проверки данных нужно
    выбирать узел предикатом, например `/collaborator/code[text() = "demo"]`.

### Равенство и подстрока

```sql
SELECT
  C.data.value(
    '(/collaborator/custom_elems/custom_elem[name = "fld_city"]/value)[1]',
    'nvarchar(100)'
  ) AS city,
  C.data.exist(
    '/collaborator/custom_elems/custom_elem[
      name = "fld_city" and string(value[1]) = "Киров"
    ]'
  ) AS city_is_kirov,
  C.data.exist(
    '/collaborator/custom_elems/custom_elem[
      name = "fld_city" and contains(string(value[1]), "иро")
    ]'
  ) AS city_contains_fragment
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

`string(value[1])` получает строковое значение типизированного `<value>`.
Сравнения строк в XQuery SQL Server чувствительны к регистру.

## Агрегация значений XQuery

`count()` подсчитывает выбранные элементы. `sum()` работает с их атомарными
значениями; явное преобразование к `xs:double` уменьшает зависимость от типа,
назначенного XML-схемой.

```sql
SELECT
  C.data.value(
    'count(/collaborator/history_states/history_state[state_id = "vacation"])',
    'int'
  ) AS vacation_count,
  C.data.value(
    'sum(
      for $comment in /collaborator/history_states/history_state[state_id = "vacation"]/comment
      return xs:double(string($comment[1]))
    )',
    'float'
  ) AS vacation_days
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Все выбранные `<comment>` должны содержать числа. Для прикладных расчётов
сложные значения удобнее сначала развернуть через `nodes()`, преобразовать
через `value()`, а затем агрегировать средствами T-SQL.

## XML как строки таблицы через `nodes()`

`nodes()` создаёт логическую копию XML для каждого выбранного узла и назначает
этому узлу контекст текущей строки.

```sql
SELECT
  C.id,
  X.node.query('.') AS custom_elem,
  X.node.value('(name)[1]', 'nvarchar(100)') AS name,
  X.node.value('(value)[1]', 'nvarchar(max)') AS value
FROM dbo.collaborator AS C
CROSS APPLY C.data.nodes(
  '/collaborator/custom_elems/custom_elem'
) AS X(node)
WHERE C.id = 1105387902724063510;
```

`query('.')` возвращает текущий `<custom_elem>`, а относительные пути
`name` и `value` читаются от него.

!!! info "`CROSS APPLY` и `OUTER APPLY`"
    Правая часть `APPLY` вычисляется для каждой строки таблицы слева и может
    использовать её колонки. Если `nodes()` возвращает пустой набор,
    `CROSS APPLY` не сохраняет исходную строку. `OUTER APPLY` сохраняет её и
    подставляет `NULL` в столбцы `X`:

    ```sql
    SELECT C.id, X.node.query('.') AS custom_elem
    FROM dbo.collaborator AS C
    OUTER APPLY C.data.nodes(
      '/collaborator/custom_elems/custom_elem'
    ) AS X(node)
    WHERE C.id = 1105387902724063510;
    ```

### Порядковый номер узла

`nodes()` не добавляет отдельный столбец с позицией. Её можно вычислить как
количество предшествующих узлов плюс один:

```sql
SELECT
  C.id,
  X.node.query('.') AS custom_elem,
  X.node.value(
    'count(
      for $item in /collaborator/custom_elems/custom_elem
      where $item << .
      return $item
    ) + 1',
    'int'
  ) AS node_number
FROM dbo.collaborator AS C
CROSS APPLY C.data.nodes(
  '/collaborator/custom_elems/custom_elem'
) AS X(node)
WHERE C.id = 1105387902724063510;
```

Оператор `<<` сравнивает положение двух узлов в документе: левый узел должен
предшествовать правому. Это оператор порядка узлов XQuery, а не побитовая
операция. `ROW_NUMBER() OVER (ORDER BY C.id)` здесь неравноценен: одинаковый
`C.id` не задаёт порядок сформированных строк.

## Практический пример: навыки сотрудника

Технические имена полей навыков начинаются с `skill_`, а включённое значение
хранится как строка `true`.

```sql
WITH custom_fields AS (
  SELECT
    C.id,
    X.node.value('(name)[1]', 'nvarchar(100)') AS field_name,
    X.node.value('(value)[1]', 'nvarchar(10)') AS field_value
  FROM dbo.collaborator AS C
  CROSS APPLY C.data.nodes(
    '/collaborator/custom_elems/custom_elem'
  ) AS X(node)
  WHERE C.id = 1105387902724063510
)
SELECT
  F.id,
  STRING_AGG(F.field_name, N', ')
    WITHIN GROUP (ORDER BY F.field_name) AS enabled_skills
FROM custom_fields AS F
WHERE F.field_name LIKE N'skill[_]%'
  AND F.field_value = N'true'
GROUP BY F.id;
```

SQL Server не поддерживает XQuery-функцию `starts-with()`. Поэтому
`nodes()` разворачивает все `<custom_elem>`, а префикс имени и значение поля
проверяются средствами T-SQL. В шаблоне `LIKE` конструкция `[_]` обозначает
литеральный символ подчёркивания. Порядок внутри `STRING_AGG()` задан явно.

## Изменение XML через `modify()`

!!! danger "Не изменяйте документную таблицу этими примерами"
    Прямой `UPDATE dbo.collaborator` выполняется вне прикладного API WebSoft и
    не рассматривается в этой статье. Примеры ниже изменяют только локальную
    XML-переменную и не записывают данные в базу.

`modify()` принимает одно выражение XML DML: `insert`, `delete` или
`replace value of`.

### Добавление дочернего узла

```sql
DECLARE @comment nvarchar(100) = N'Тестовый отпуск';
DECLARE @document xml = N'
<collaborator>
  <history_states/>
</collaborator>';

SET @document.modify('
  insert
    <history_state>
      <id>state_1</id>
      <state_id>vacation</state_id>
      <comment>{sql:variable("@comment")}</comment>
    </history_state>
  as last into (/collaborator/history_states)[1]
');

SELECT @document AS modified_document;
```

`as first into` и `as last into` добавляют первый или последний дочерний узел.
Операторы `before` и `after` вставляют соседний узел относительно выбранного.
`sql:variable()` передаёт в XML DML значение переменной T-SQL.

### Замена значения

```sql
DECLARE @comment nvarchar(100) = N'Обновлённый комментарий';
DECLARE @document xml = N'
<collaborator>
  <comment>Исходный комментарий</comment>
</collaborator>';

SET @document.modify('
  replace value of
    (/collaborator/comment/text())[1]
  with sql:variable("@comment")
');

SELECT @document AS modified_document;
```

Цель `replace value of` должна быть одиночным узлом. Поэтому путь заканчивается
позиционным предикатом `[1]`.

### Удаление пустых элементов

```sql
DECLARE @document xml = N'
<collaborator>
  <custom_elems>
    <custom_elem/>
    <custom_elem><name>fld_city</name><value>Киров</value></custom_elem>
    <custom_elem/>
  </custom_elems>
</collaborator>';

SET @document.modify('
  delete /collaborator/custom_elems/custom_elem[not(node())]
');

SELECT @document AS modified_document;
```

Предикат `not(node())` выбирает `<custom_elem>` без дочерних узлов. `delete`
удаляет все элементы, попавшие в выбранную последовательность.

## Ограничения

- `value()` требует выражение, которое статически возвращает не более одного
  значения; обычно для этого используется внешнее `[1]`.
- `exist()` проверяет непустую последовательность, а не логическое значение
  выражения.
- SQL Server поддерживает не все функции XQuery; в частности, функции
  `starts-with()` нет в реализованном наборе.
- Поведение статической типизации зависит от коллекции XML-схем, связанной с
  колонкой.
- `nodes()` создаёт логические копии с новым контекстным узлом; применять к ним
  `modify()` нельзя.
- `CROSS APPLY` исключает исходную строку при пустом результате справа;
  `OUTER APPLY` сохраняет её.
- Для стандартных полей и массовой фильтрации следует использовать таблицу
  каталога, а не повторно извлекать значения из `data`.
- Прямое изменение документной таблицы SQL-запросом требует отдельной проверки
  согласованности с механизмами WebSoft.

## Документация Microsoft

- [Методы типа `xml`](https://learn.microsoft.com/ru-ru/sql/t-sql/xml/xml-data-type-methods?view=sql-server-ver16)
- [Метод `nodes()`](https://learn.microsoft.com/ru-ru/sql/t-sql/xml/nodes-method-xml-data-type?view=sql-server-ver16)
- [Метод `modify()`](https://learn.microsoft.com/ru-ru/sql/t-sql/xml/modify-method-xml-data-type?view=sql-server-ver16)
- [Типизированный и нетипизированный XML](https://learn.microsoft.com/ru-ru/sql/relational-databases/xml/compare-typed-xml-to-untyped-xml?view=sql-server-ver16)
- [Статическая типизация XQuery](https://learn.microsoft.com/ru-ru/sql/xquery/xquery-and-static-typing?view=sql-server-ver16)
- [Функции доступа `data()`, `string()` и `text()`](https://learn.microsoft.com/en-us/sql/xquery/data-accessor-functions?view=sql-server-ver16)
- [Поддерживаемые функции XQuery](https://learn.microsoft.com/en-us/sql/xquery/xquery-functions-against-the-xml-data-type?view=sql-server-ver16)
