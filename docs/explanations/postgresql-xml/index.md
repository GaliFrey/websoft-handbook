# XML-поля WebSoft HCM в PostgreSQL

Страница показывает, как читать XML-документы объектов WebSoft HCM прямыми
SQL-запросами к PostgreSQL. Примеры относятся к WebSoft HCM Server
`2023.2.906`, PostgreSQL provider `1.24.4.18` и PostgreSQL `16.3`.

## Каталог и XML-документ

Для сотрудника используются две таблицы:

| Таблица | Назначение |
| --- | --- |
| `dbo.collaborators` | Каталог с основными полями, пригодными для обычной выборки и фильтрации |
| `dbo.collaborator` | Документная таблица с полным XML в колонке `data` |

Если значение уже присутствует в `dbo.collaborators`, следует читать его из
каталога. Обращение к `dbo.collaborator.data` оправдано для данных, которых нет
в каталоге: например, повторяющихся или кастомных полей.

Во всех запросах используется ID тестового сотрудника
`1105387902724063510`.

## Основные функции

PostgreSQL обрабатывает выражения в `xpath()` и `xpath_exists()` как XPath 1.0.
Аргумент `data` должен содержать XML-документ с одним корневым элементом.

### `xpath()`

`xpath()` возвращает массив `xml[]`. Если XPath возвращает скалярное значение,
результатом всё равно будет массив из одного элемента.

```sql
SELECT xpath('/collaborator/category_id', C.data) AS category_nodes
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### `xpath_exists()`

`xpath_exists()` возвращает `true`, если выражение XPath даёт результат, отличный
от пустого набора узлов.

```sql
SELECT
  xpath_exists('/collaborator/category_id', C.data) AS category_exists,
  xpath_exists('/collaborator/category_id/text()', C.data) AS category_value_exists
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### `unnest()`

`unnest()` разворачивает массив в набор строк. С `WITH ORDINALITY` функция также
возвращает порядковый номер элемента, начиная с 1.

```sql
SELECT C.id, X.node_number, X.category_node
FROM dbo.collaborator AS C
CROSS JOIN LATERAL unnest(
  xpath('/collaborator/category_id', C.data)
) WITH ORDINALITY AS X(category_node, node_number)
WHERE C.id = 1105387902724063510
ORDER BY X.node_number;
```

!!! info "Зачем нужен `LATERAL`"
    `LATERAL` позволяет элементу справа в секции `FROM` обращаться к колонкам
    таблицы слева. В этом запросе `unnest()` использует `C.data` текущей строки
    `dbo.collaborator`. Для каждой строки `C` сначала вычисляется `xpath()`,
    затем полученный массив разворачивается в строки, которые `CROSS JOIN`
    соединяет с исходной строкой.

    Для табличных функций PostgreSQL ключевое слово `LATERAL` необязательно:
    их аргументы могут ссылаться на предшествующие элементы `FROM` и без него.
    Здесь оно оставлено, чтобы явно показать зависимость функции от `C.data`.

    Если `xpath()` возвращает пустой массив, `unnest()` не создаёт строк и
    исходная строка не попадает в результат `CROSS JOIN`. Чтобы сохранить её с
    `NULL` в столбцах `X`, используется `LEFT JOIN LATERAL ... ON true`:

    ```sql
    SELECT C.id, X.node_number, X.category_node
    FROM dbo.collaborator AS C
    LEFT JOIN LATERAL unnest(
      xpath('/collaborator/category_id', C.data)
    ) WITH ORDINALITY AS X(category_node, node_number) ON true
    WHERE C.id = 1105387902724063510
    ORDER BY X.node_number;
    ```

    Подробнее: [подзапросы `LATERAL` в документации PostgreSQL](https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-LATERAL).

## Выбор XML-узлов

### Первый узел

Позиции узлов в XPath и стандартные индексы массивов PostgreSQL начинаются с 1.
Следующие выражения показывают разницу между фильтрацией в XPath и получением
элемента уже сформированного массива:

```sql
SELECT
  xpath('/collaborator/lastname[1]', C.data) AS first_node_array,
  xpath('/collaborator/lastname[2]', C.data) AS second_node_array,
  (xpath('/collaborator/lastname', C.data))[1] AS first_node
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Первые два столбца имеют тип `xml[]`. Третий столбец содержит один `xml`-узел
либо `NULL`, если массив пуст.

### Узел вместе с содержимым

```sql
SELECT xpath('/collaborator/custom_elems', C.data) AS custom_elems
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Запрос возвращает элементы `<custom_elems>` целиком, включая вложенные
`<custom_elem>`.

### Узел по позиции

```sql
SELECT
  xpath(
    '/collaborator/custom_elems/custom_elem[6]',
    C.data
  ) AS sixth_node_array,
  (
    xpath('/collaborator/custom_elems/custom_elem', C.data)
  )[6] AS sixth_node
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Первый столбец имеет тип `xml[]`, второй — `xml`.

### Последние узлы

Когда количество соседних узлов заранее неизвестно, XPath-функция `last()`
позволяет обратиться к элементам с конца:

```sql
SELECT
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[last()]/name/text()',
      C.data
    )
  )[1]::text AS last_name,
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[last() - 1]/name/text()',
      C.data
    )
  )[1]::text AS previous_name,
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[last() - 2]/name/text()',
      C.data
    )
  )[1]::text AS third_from_end_name
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

## Отбор по вложенным значениям

### Родительский узел по значению дочернего

```sql
SELECT xpath(
  '/collaborator/custom_elems/custom_elem[name = "fld_city"]',
  C.data
) AS city_custom_elem
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Условие в квадратных скобках отбирает `<custom_elem>`, внутри которого есть
`<name>fld_city</name>`.

### Значение соседнего узла

```sql
SELECT xpath(
  '/collaborator/custom_elems/custom_elem[name = "fld_city"]/value',
  C.data
) AS city_value_node
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

После отбора родительского `<custom_elem>` путь продолжается до соседнего
элемента `<value>`.

### Равенство и подстрока

```sql
SELECT
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[name = "fld_city"]/value/text()',
      C.data
    )
  )[1]::text AS city,
  xpath_exists(
    '/collaborator/custom_elems/custom_elem[name = "fld_city" and value = "Киров"]',
    C.data
  ) AS city_is_kirov,
  xpath_exists(
    '/collaborator/custom_elems/custom_elem[name = "fld_city" and contains(value, "иро")]',
    C.data
  ) AS city_contains_fragment
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

`contains(value, "иро")` проверяет строковое значение именно элемента
`<value>`. Выражение `contains(., "иро")` проверяло бы объединённое текстовое
содержимое всего текущего `<custom_elem>` и было бы менее точным.

## Получение значений и приведение типов

Чтобы получить содержимое элемента без XML-разметки, путь должен заканчиваться
на `text()`. После выбора первого элемента массива значение можно привести к
нужному типу PostgreSQL.

```sql
SELECT
  (xpath('/collaborator/id/text()', C.data))[1]::text::bigint AS xml_id,
  (xpath('/collaborator/lastname/text()', C.data))[1]::text AS lastname,
  (xpath('/collaborator/birth_date/text()', C.data))[1]::text::date AS birth_date,
  (xpath('/collaborator/is_dismiss/text()', C.data))[1]::text::boolean AS is_dismiss
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Неверное приведение завершается ошибкой PostgreSQL. Например, фамилия не может
быть преобразована в дату:

```sql
SELECT (xpath('/collaborator/lastname/text()', C.data))[1]::text::date
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### Значения кастомных полей

```sql
SELECT
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[name = "fld_city"]/value/text()',
      C.data
    )
  )[1]::text AS city,
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[name = "fld_birth_date"]/value/text()',
      C.data
    )
  )[1]::text::timestamp AS custom_birth_date,
  (
    xpath(
      '/collaborator/custom_elems/custom_elem[name = "fld_skills"]/value/text()',
      C.data
    )
  )[1]::text AS skills
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Для значения с часовым смещением, например
`2024-10-01T00:00:00+00:00`, следует использовать `timestamptz`. Приведение к
`timestamp` без часового пояса не сохраняет смещение.

## Повторяющиеся узлы

### Массив типизированных значений

Если XPath возвращает несколько текстовых узлов, тип нужно менять у массива
целиком:

```sql
SELECT
  xpath(
    '/collaborator/history_states/history_state[state_id = "vacation"]/start_date/text()',
    C.data
  )::text[]::timestamptz[] AS vacation_starts,
  xpath(
    '/collaborator/history_states/history_state[state_id = "vacation"]/finish_date/text()',
    C.data
  )::text[]::timestamptz[] AS vacation_finishes
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### Агрегация средствами XPath

XPath 1.0 предоставляет функции `count()` и `sum()`. В этом примере содержимое
`<comment>` должно состоять только из чисел:

```sql
SELECT
  (
    xpath(
      'sum(/collaborator/history_states/history_state[state_id = "vacation"]/comment/text())',
      C.data
    )
  )[1]::text::integer AS vacation_days,
  (
    xpath(
      'count(/collaborator/history_states/history_state[state_id = "vacation"]/comment/text())',
      C.data
    )
  )[1]::text::integer AS vacation_count
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

Для прикладных расчётов над сложными значениями надёжнее сначала получить
типизированные строки через `XMLTABLE`, а затем использовать агрегаты SQL.

### Отсутствующий, пустой и заполненный узел

Самодостаточный запрос показывает разницу между отсутствием элемента, пустым
элементом и элементом с текстом:

```sql
WITH samples(label, data) AS (
  VALUES
    ('absent', '<collaborator/>'::xml),
    ('empty', '<collaborator><code/></collaborator>'::xml),
    ('filled', '<collaborator><code>demo</code></collaborator>'::xml)
)
SELECT
  S.label,
  xpath('/collaborator/code', S.data) AS code_node,
  xpath_exists('/collaborator/code', S.data) AS node_exists,
  xpath_exists('/collaborator/code/text()', S.data) AS text_exists
FROM samples AS S
ORDER BY S.label;
```

## XML как строки таблицы

### `unnest()` и повторный `xpath()`

```sql
SELECT
  X.node_number,
  (xpath('/custom_elem/name/text()', X.node))[1]::text AS name,
  (xpath('/custom_elem/value/text()', X.node))[1]::text AS value
FROM dbo.collaborator AS C
CROSS JOIN LATERAL unnest(
  xpath('/collaborator/custom_elems/custom_elem', C.data)
) WITH ORDINALITY AS X(node, node_number)
WHERE C.id = 1105387902724063510
ORDER BY X.node_number;
```

`WITH ORDINALITY` сохраняет позицию, с которой элемент был выдан функцией
`unnest()`. Оконная функция с сортировкой только по `C.id` не может надёжно
восстановить порядок соседних XML-узлов.

### `XMLTABLE`

Когда из каждого повторяющегося узла нужны несколько типизированных полей,
`XMLTABLE` выражает преобразование напрямую:

```sql
SELECT X.node_number, X.name, X.value
FROM dbo.collaborator AS C
CROSS JOIN LATERAL XMLTABLE(
  '/collaborator/custom_elems/custom_elem'
  PASSING C.data
  COLUMNS
    node_number FOR ORDINALITY,
    name text PATH 'name',
    value text PATH 'value'
) AS X
WHERE C.id = 1105387902724063510
ORDER BY X.node_number;
```

## Практический пример: навыки сотрудника

Технические имена полей навыков начинаются с `skill_`, а включённое значение
хранится как строка `true`.

!!! warning "Сравнивайте текст, а не наличие узла"
    Условие XPath `value = true()` проверяет булево представление набора узлов.
    Любой существующий `<value>`, включая `<value>false</value>`, делает такой
    набор истинным. Для проверки содержимого используйте `value = "true"`.

### Технические имена полей

```sql
SELECT array_to_string(
  xpath(
    '/collaborator/custom_elems/custom_elem[starts-with(name, "skill_") and value = "true"]/name/text()',
    C.data
  )::text[],
  ', '
) AS enabled_skill_names
FROM dbo.collaborator AS C
WHERE C.id = 1105387902724063510;
```

### Отображаемые названия полей

Сначала запрос читает имена и заголовки кастомных полей из шаблона, затем
соединяет их с включёнными навыками сотрудника:

```sql
WITH custom_template AS (
  SELECT X.name, X.title
  FROM dbo."(spxml_blobs)" AS B
  CROSS JOIN LATERAL XMLTABLE(
    '/custom_templates/collaborator/fields/field'
    PASSING XMLPARSE(DOCUMENT convert_from(B.data, 'UTF-8'))
    COLUMNS
      name text PATH 'name',
      title text PATH 'title'
  ) AS X
  WHERE B.url = 'x-local://wt_data/lists/wtv_custom_templates.xml'
),
skills AS (
  SELECT C.id, X.skill_name
  FROM dbo.collaborator AS C
  CROSS JOIN LATERAL XMLTABLE(
    '/collaborator/custom_elems/custom_elem[starts-with(name, "skill_") and value = "true"]'
    PASSING C.data
    COLUMNS skill_name text PATH 'name'
  ) AS X
  WHERE C.id = 1105387902724063510
)
SELECT
  S.id,
  string_agg(T.title, ', ' ORDER BY T.title) AS enabled_skills
FROM skills AS S
INNER JOIN custom_template AS T ON T.name = S.skill_name
GROUP BY S.id;
```

В `dbo."(spxml_blobs)"` колонка `data` имеет тип `bytea`. Функция
`convert_from()` декодирует байты UTF-8 в `text`, после чего
`XMLPARSE(DOCUMENT ...)` преобразует текст в XML-документ для `XMLTABLE`.
Сокращённая форма `convert_from(...)::xml` непосредственно после `PASSING`
на `906_PSQL` завершается синтаксической ошибкой `42601`.

Порядок внутри `string_agg()` задан явно. Без `ORDER BY` порядок объединённых
значений не гарантирован.

## Ограничения

- XPath-функции PostgreSQL поддерживают XPath 1.0, а не XQuery и не XPath 2.0+.
- `xpath()` и `xpath_exists()` принимают XML-документ с одним корневым
  элементом, а не произвольный XML-фрагмент.
- `xpath()` возвращает `xml[]` даже для скалярного результата.
- Пустой массив, отсутствующий узел и пустой текстовый узел — разные случаи.
- Порядок строк нужно задавать явно. Порядок элементов набора узлов XPath 1.0
  формально не гарантирован.
- Обработка `data` выполняется для каждой выбранной строки. Для стандартных
  полей и массовой фильтрации следует использовать таблицу каталога.

## Документация PostgreSQL

- [XML-функции PostgreSQL 16](https://www.postgresql.org/docs/16/functions-xml.html)
- [Массивы PostgreSQL 16](https://www.postgresql.org/docs/16/functions-array.html)
- [Табличные функции и `WITH ORDINALITY`](https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-TABLEFUNCTIONS)
- [Агрегатные функции](https://www.postgresql.org/docs/16/functions-aggregate.html)
