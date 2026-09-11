# Как XQuery превращается в SQL: WebSoft HCM 906 и MSSQL

Примеры получены на WebSoft HCM Server `2023.2.906` с MSSQL provider
`1.24.4.18` и Microsoft SQL Server `16.0.4275.2`.

`LuceneFTIndex` и `XQueryTemplateCache` были отключены. Для проверки обычного
`doc-contains()` использовался заполненный полнотекстовый индекс MSSQL на
`dbo.collaborator(data)`.

В SQL встречаются параметры `@p0`, `@p1` и далее. Их значения указаны после
примеров. Прикладной код должен выполнять XQuery, а не копировать полученный
SQL: внутренняя реализация может измениться в другой сборке.

## Получение полей

### Поля через Fields()

**XQuery:**

```xquery
for $elem in collaborators
return $elem/Fields('id', 'fullname', 'position_name')
```

**SQL:**

```sql
select
  t_elem.[id],
  t_elem.[fullname],
  t_elem.[position_name]
from dbo.[collaborators] t_elem
```

Записи `Fields(id, fullname, position_name)` и
`$elem/id, $elem/fullname, $elem/position_name` сформировали такой же SQL.
Кавычки внутри `Fields()` результат не изменили.

### Возврат всей записи

**XQuery:**

```xquery
for $elem in collaborators
where $elem/id = 1105387902724063510
return $elem
```

`return $elem` не превращается в `select *`. Provider явно перечисляет все
55 доступных полей плоского каталога. Поля дат дополнительно преобразуются в
строки:

**SQL:**

```sql
select
  t_elem.[id],
  t_elem.[code],
  t_elem.[fullname],
  -- остальные поля каталога
  convert(varchar(19),t_elem.[hire_date],127) [hire_date],
  -- остальные поля каталога
  t_elem.[app_instance_id]
from dbo.[collaborators] t_elem
where t_elem.[id]=@p0
```

Параметр: `@p0 BigInt = 1105387902724063510`. Состав `SELECT` зависит от
схемы каталога конкретной сборки.

### Псевдонимы полей

**XQuery:**

```xquery
for $elem in collaborators
return
  $elem/id collaborator_id,
  $elem/fullname collaborator_name
```

**SQL:**

```sql
select
  t_elem.[id][collaborator_id],
  t_elem.[fullname][collaborator_name]
from dbo.[collaborators] t_elem
```

В MSSQL конструкция `[id][collaborator_id]` корректна: второй идентификатор —
псевдоним поля. Ключевое слово `AS` для него необязательно.

## Условия

### Число

**XQuery:**

```xquery
for $elem in collaborators
where $elem/id = 1105387902724063510
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[id]=@p0
```

Параметр: `@p0 BigInt = 1105387902724063510`.

Остальные операторы сравнения также параметризуются:

| Условие XQuery | Условие SQL |
| --- | --- |
| `$elem/id != 1105387902724063510` | `t_elem.[id]<>@p0 or t_elem.[id] is null` |
| `$elem/id > 1105387902724063510` | `t_elem.[id]>@p0` |
| `$elem/id >= 1105387902724063510` | `t_elem.[id]>=@p0` |
| `$elem/id < 1105387902724063510` | `t_elem.[id]<@p0` |
| `$elem/id <= 1105387902724063510` | `t_elem.[id]<=@p0` |

Во всех пяти запросах параметр имеет тип `BigInt`. Для `!=` provider
добавляет `or ... is null`, как и для других отрицаний.

### Строка

**XQuery:**

```xquery
for $elem in collaborators
where $elem/code = '13744'
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where t_elem.[code]=@p0
```

Параметр: `@p0 VarChar = 13744`.

### Поиск подстроки через contains()

**XQuery:**

```xquery
for $elem in collaborators
where contains($elem/fullname, 'Анисимов')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[fullname] like @p0
```

Параметр уже содержит маску: `@p0 VarChar = %Анисимов%`.

Специальные символы обрабатываются неодинаково:

| Аргумент `contains()` | Значение `@p0` |
| --- | --- |
| `%` | `%%%` |
| `_` | `%[_]%` |
| `[` | `%[%` |

Подчёркивание экранируется как литерал, `%` остаётся SQL-маской, а открывающая
квадратная скобка передаётся в `LIKE` без дополнительного экранирования.

Отрицание включает строки с `NULL`:

**XQuery:**

```xquery
for $elem in collaborators
where contains($elem/fullname, 'Анисимов') = false()
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where not(t_elem.[fullname] like @p0)
   or t_elem.[fullname] is null
```

Параметр уже содержит маску: `@p0 VarChar = %Анисимов%`.

### false(), true(), null(), пустая строка и IsEmpty()

#### false()

**XQuery:**

```xquery
for $elem in collaborators
where $elem/is_dismiss = false()
return $elem/Fields('id', 'is_dismiss')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[is_dismiss]
from dbo.[collaborators] t_elem
where t_elem.[is_dismiss]=0
   or t_elem.[is_dismiss] is null
```

Отрицание:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/is_dismiss != false()
return $elem/Fields('id', 'is_dismiss')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[is_dismiss]
from dbo.[collaborators] t_elem
where t_elem.[is_dismiss]<>0
```

Равенство `false()` включает `NULL`, а отрицание `false()` — только записи со значением `true`.

#### true()

**XQuery:**

```xquery
for $elem in collaborators
where $elem/is_dismiss = true()
return $elem/Fields('id', 'is_dismiss')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[is_dismiss]
from dbo.[collaborators] t_elem
where t_elem.[is_dismiss]=1
```

Отрицание:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/is_dismiss != true()
return $elem/Fields('id', 'is_dismiss')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[is_dismiss]
from dbo.[collaborators] t_elem
where t_elem.[is_dismiss]<>1
   or t_elem.[is_dismiss] is null
```

В отличие от `!= false()`, условие `!= true()` включает `NULL`.

#### null()

**XQuery:**

```xquery
for $elem in collaborators
where $elem/position_id = null()
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_id]
from dbo.[collaborators] t_elem
where t_elem.[position_id] is null
```

Отрицание:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/position_id != null()
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_id]
from dbo.[collaborators] t_elem
where t_elem.[position_id] is not null
```

#### Пустая строка

**XQuery:**

```xquery
for $elem in collaborators
where $elem/code = ''
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where t_elem.[code] is null
```

Отрицание:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/code != ''
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where t_elem.[code] is not null
```

На этом provider пустая строка в сравнении с полем трактуется как `NULL`.

#### IsEmpty()

**XQuery:**

```xquery
for $elem in collaborators
where IsEmpty($elem/birth_date) = true()
return $elem/Fields('id', 'birth_date')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[birth_date]
from dbo.[collaborators] t_elem
where (case when t_elem.[birth_date] is null then 1 else 0 end)=1
```

Отрицание:

**XQuery:**

```xquery
for $elem in collaborators
where IsEmpty($elem/birth_date) = false()
return $elem/Fields('id', 'birth_date')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[birth_date]
from dbo.[collaborators] t_elem
where (case when t_elem.[birth_date] is null then 1 else 0 end)=0
   or (case when t_elem.[birth_date] is null then 1 else 0 end) is null
```

Форма `IsEmpty($elem/birth_date) != true()` также работает. Она заменяет
`=0` на `<>1`; остальная конструкция SQL не меняется.

### Дата

**XQuery:**

```xquery
for $elem in collaborators
where $elem/hire_date = date('2010-03-17')
return $elem/Fields('id', 'hire_date')
```

**SQL:**

```sql
SET DATEFORMAT dmy;
select t_elem.[id],t_elem.[hire_date]
from dbo.[collaborators] t_elem
where t_elem.[hire_date]=@p0
```

Параметр: `@p0 DateTime = 2010-03-17T00:00:00`.

Диапазон дат:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/hire_date >= date('2010-03-01')
  and $elem/hire_date < date('2010-04-01')
return $elem/Fields('id', 'hire_date')
```

**SQL:**

```sql
SET DATEFORMAT dmy;
select t_elem.[id],t_elem.[hire_date]
from dbo.[collaborators] t_elem
where t_elem.[hire_date]>=@p0
  and t_elem.[hire_date]<@p1
```

Параметры: `@p0 DateTime = 2010-03-01T00:00:00`,
`@p1 DateTime = 2010-04-01T00:00:00`.

### Группировка and и or

**XQuery:**

```xquery
for $elem in collaborators
where
  ($elem/is_dismiss = false()
    and contains($elem/fullname, 'Анисимов'))
  or $elem/id = 1105387902724063521
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where (
        (t_elem.[is_dismiss]=0 or t_elem.[is_dismiss] is null)
        and t_elem.[fullname] like @p0
      )
   or t_elem.[id]=@p1
```

Параметры:

- `@p0 VarChar = %Анисимов%`;
- `@p1 BigInt = 1105387902724063521`.

Скобки сохранили порядок вычисления условий.

## Несколько каталогов

### Перечисление каталогов через запятую

**XQuery:**

```xquery
for
  $elem in collaborators,
  $pos in positions
where
  $elem/position_id = $pos/id
  and $elem/id = 1105387902724063510
return
  $elem/id,
  $elem/fullname,
  $pos/name
```

**SQL:**

```sql
select
  t_elem.[id],
  t_elem.[fullname],
  t_pos.[name]
from dbo.[positions] t_pos,dbo.[collaborators] t_elem
where t_elem.[position_id]=t_pos.[id]
  and t_elem.[id]=@p0
```

Параметр: `@p0 BigInt = 1105387902724063510`.

Связь каталогов осталась в `WHERE`; отдельный `JOIN` не создаётся.

### Связь через some/satisfies

**XQuery:**

```xquery
for $elem in collaborators
where
  some $pos in positions satisfies
    ($elem/position_id = $pos/id)
  and $pos/name = 'Руководитель отдела обучения и развития'
return $elem/Fields('id', 'position_name')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_name]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select t_elem.[id]
  from dbo.[collaborators] t_elem
  inner join dbo.[positions] t_pos
    on t_elem.[position_id]=t_pos.[id]
  where t_pos.[name]=@p0
)
```

Параметр: `@p0 VarChar = Руководитель отдела обучения и развития`.

Условие в скобках после `satisfies` стало условием `JOIN`, а проверка имени —
условием вложенного `WHERE`.

Для трёх каталогов используется та же схема:

**XQuery:**

```xquery
for $elem in collaborators
where
  some $app in appointment_types satisfies
    ($pos/position_appointment_type_id = $app/id)
  and some $pos in positions satisfies
    ($elem/position_id = $pos/id)
  and $app/code = 'main'
return $elem/Fields('id', 'position_name')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_name]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select t_elem.[id]
  from dbo.[collaborators] t_elem
  inner join dbo.[positions] t_pos
    on t_elem.[position_id]=t_pos.[id]
  inner join dbo.[appointment_types] t_app
    on t_pos.[position_appointment_type_id]=t_app.[id]
  where t_app.[code]=@p0
)
```

Параметр: `@p0 VarChar = main`.

### Операторы join, ljoin и rjoin

MSSQL provider `1.24.4.18` поддерживает явные операторы соединения. Это
отличие от provider `1.22.6.8` в сборке 434, где следующие запросы завершались
до формирования SQL.

#### join

**XQuery:**

```xquery
for $pos in positions join $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

**SQL:**

```sql
select t_elem.[id],t_pos.[name]
from dbo.[collaborators] t_elem
inner join dbo.[positions] t_pos
  on t_elem.[position_id]=t_pos.[id]
where t_elem.[id]=@p0
```

#### ljoin

**XQuery:**

```xquery
for $pos in positions ljoin $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

**SQL:**

```sql
select t_elem.[id],t_pos.[name]
from dbo.[collaborators] t_elem
left join dbo.[positions] t_pos
  on t_elem.[position_id]=t_pos.[id]
where t_elem.[id]=@p0
```

#### rjoin

**XQuery:**

```xquery
for $pos in positions rjoin $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

**SQL:**

```sql
select t_elem.[id],t_pos.[name]
from dbo.[collaborators] t_elem
right join dbo.[positions] t_pos
  on t_elem.[position_id]=t_pos.[id]
where t_elem.[id]=@p0
```

Во всех трёх случаях параметр: `@p0 BigInt = 1105387902724063510`.

Условие по `t_elem.id` в `WHERE` отбрасывает строки правой таблицы без пары,
поэтому именно этот пример с `RIGHT JOIN` по результату не показывает все
позиции без сотрудника.

#### Антисоединение через ljoin и MatchSome()

**XQuery:**

```xquery
for $excluded in collaborators ljoin $elem in collaborators
  on $elem/id = $excluded/id
    and MatchSome($excluded/code, ('13744', '109295'))
where $excluded/id = null()
return $elem/Fields('id')
```

**SQL:**

```sql
select t_elem.[id]
from dbo.[collaborators] t_elem
left join dbo.[collaborators] t_excluded
  on t_elem.[id]=t_excluded.[id]
  and t_excluded.[code] in (@p0,@p1)
where t_excluded.[id] is null
```

Параметры: `@p0 VarChar = 13744`, `@p1 VarChar = 109295`.

Условие по правой таблице остаётся в `ON`, а проверка `null()` превращается в
`IS NULL`. В результате запрос исключает сотрудников с указанными кодами.

## MatchSome()

Положительный вариант превращается в `IN`:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where t_elem.[code] in (@p0,@p1)
```

Параметры: `@p0 VarChar = 13744`, `@p1 VarChar = 109295`.

Для настоящего множественного поля provider использует XML-метод `exist()`,
а не обычный `IN`. Например, `collaborators.category_id` описан как
множественное поле:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome(
  $elem/category_id,
  ('123', '456')
)
return $elem/Fields('id', 'fullname', 'category_id')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname],t_elem.[category_id]
from dbo.[collaborators] t_elem
where (t_elem.[category_id].exist('category_id[.=("123","456")]')=1)
```

Значения встроены в XQuery-выражение XML-метода. Provider также создаёт
параметры `@p0 VarChar = 123` и `@p1 VarChar = 456`, но в SQL нет
соответствующих плейсхолдеров, поэтому команда их не использует.

Отрицание через `false()` превращается в невалидный MSSQL:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295')) = false()
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where (((t_elem.[code] in (@p0,@p1))=0)
OR (((t_elem.[code] in (@p0,@p1)))  IS NULL))
```

Параметры: `@p0 VarChar = 13744`, `@p1 VarChar = 109295`.

MSSQL не разрешает сравнивать предикат `IN` с числом. Две формы с `not` также
не работают:

**XQuery:**

```xquery
for $elem in collaborators
where not MatchSome($elem/code, ('13744', '109295'))
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where [not](t_elem.[code] in (@p0,@p1))
```

и

**XQuery:**

```xquery
for $elem in collaborators
where not(MatchSome($elem/code, ('13744', '109295')))
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[code]
from dbo.[collaborators] t_elem
where not()
```

Все три формы отрицания создают невалидный SQL. На сборке 906 их использовать
нельзя.

## ForeignElem()

**XQuery:**

```xquery
for $elem in collaborators
where ForeignElem($elem/position_id)/name = 'Бизнес-тренер'
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_id]
from dbo.[collaborators] t_elem
left join dbo.[positions] [f1560031482]
  on t_elem.[position_id]=[f1560031482].[id]
where [f1560031482].[name]=@p0
```

Параметр: `@p0 VarChar = Бизнес-тренер`.

`ForeignElem()` стал `LEFT JOIN`. Следующее условие по имени связанной позиции
находится в `WHERE`, поэтому строки без позиции всё равно отбрасываются. Число
в служебном псевдониме `f1560031482` может меняться между запусками, поэтому
на него нельзя ссылаться в прикладном коде.

## ForeignDispName()

Форма вызова как атрибута, приведённая в документации, не работает в MSSQL
provider `1.24.4.18`:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/position_id/ForeignDispName = 'Бизнес-тренер'
return $elem/Fields('id', 'position_id')
```

Запрос завершается ошибкой `Invalid column name 'ForeignDispName'`. В
сортировке получается следующий SQL:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
order by $elem/position_id/ForeignDispName
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[position_id]
from dbo.[collaborators] t_elem
where t_elem.[code] in (@p0,@p1)
order by t_elem.[position_id].[ForeignDispName] asc
```

Этот SQL невалиден для MSSQL: `position_id` имеет тип `BigInt`, поэтому
прямой запуск завершается ошибкой `Cannot call methods on bigint`. Вызов
`ForeignDispName($elem/position_id)` как обычной функции тоже неприменим:
provider оставляет `ForeignDispName(...)` в SQL, а такой SQL-функции нет. Для
получения имени связанного объекта на этой сборке нужно использовать
`ForeignElem()` или явное соединение каталогов.

## doc-contains()

### Обычный поиск по тексту документа

**XQuery:**

```xquery
for $elem in collaborators
where doc-contains($elem/id, '', 'Анисимов')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select [id]
  from dbo.[collaborator]
  where contains(*,'"Анисимов*"')
)
```

!!! warning "Для обычного doc-contains нужен полнотекстовый индекс MSSQL"
    При `LuceneFTIndex=False` функция использует SQL Server `CONTAINS`.
    Внешняя выборка идёт из плоского каталога `dbo.collaborators`, но поиск
    выполняется в документной таблице `dbo.collaborator`. Полнотекстовый
    индекс нужен на её XML-колонке `data`.

    Без полнотекстового индекса этот SQL не выполняется.

### Кастомное поле

Равенство кастомного поля:

**XQuery:**

```xquery
for $elem in collaborators
where doc-contains($elem/id, '', '[f_zsbu = зеленый]')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select [id]
  from dbo.[collaborator]
  where data.exist('collaborator/custom_elems/custom_elem[name = "f_zsbu" and value[1]="зеленый"]') = 1
)
```

Поиск подстроки:

**XQuery:**

```xquery
for $elem in collaborators
where doc-contains($elem/id, '', '[f_zsbu contains зелен]')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select [id]
  from dbo.[collaborator]
  where data.exist('collaborator/custom_elems/custom_elem[name = "f_zsbu" and contains(value[1],"зелен")]') = 1
)
```

Булево значение:

**XQuery:**

```xquery
for $elem in collaborators
where doc-contains($elem/id, '', '[is_universal = true~bool]')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[id] in (
  select [id]
  from dbo.[collaborator]
  where data.exist('collaborator/custom_elems/custom_elem[name = "is_universal" and xs:boolean(value[1])=true()]') = 1
)
```

## Иерархия

!!! warning "Форматирование XQuery меняет результат"
    На сборке 906 legacy-препроцессор разбирает иерархический запрос как
    строку и чувствителен к пробелам и переносам. Однострочная и многострочная
    записи одного выражения могут сформировать разный итоговый XQuery.

    Используйте одну из проверенных ниже точных форм и не переформатируйте её
    без повторной проверки на целевой сборке. Ограничение не относится к
    обычным XQuery без иерархической обработки.

!!! danger "Не используйте IsHierChild без order by Hier()"
    В `tools.xquery()` сборки 906 конструкция `order by $elem/Hier()` не
    ограничивается сортировкой. Legacy-препроцессор удаляет условие
    `IsHierChild()` или `IsHierChildOrSelf()` и переносит его ID и режим в
    `/Hier(ID, '-')` или `/Hier(ID, '+')`.

    Если `/Hier()` отсутствует, условие удаляется без эквивалентной замены.
    Запрос успешно выполняется, но потенциально возвращает весь каталог.

Опасный пример:

**XQuery:**

```xquery
for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) return $elem/Fields('id', 'name')
```

Фактически provider получает запрос:

**XQuery:**

```xquery
for $elem in subdivisions return $elem/Fields('id', 'name')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[name]
from dbo.[subdivisions] t_elem
```

То же преобразование происходит с `IsHierChildOrSelf()`.

!!! danger "Учитывайте одновременно порядок условий и переносы"
    Надёжная однострочная форма ставит `IsHierChild()` первым условием после
    `where`:

    **XQuery:**

    ```xquery
    for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) and $elem/is_disbanded = false() order by $elem/Hier() return $elem/Fields('id', 'name')
    ```

    После переноса иерархии остаётся корректное условие:

    **XQuery:**

    ```xquery
    for $elem in subdivisions where $elem/is_disbanded = false() order by $elem/Hier(  6327975429225669221,'-') return $elem/Fields('id', 'name')
    ```

    Обратный порядок в той же однострочной записи удаляет `where` и оставляет
    лишний `and`:

    **XQuery:**

    ```xquery
    for $elem in subdivisions where $elem/is_disbanded = false() and IsHierChild($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
    ```

    Препроцессор формирует невалидный XQuery без `where` и с лишним `and`:

    **XQuery:**

    ```xquery
    for $elem in subdivisions $elem/is_disbanded = false() and order by $elem/Hier(  6327975429225669221,'-') return $elem/Fields('id', 'name')
    ```

    Однако многострочная запись с обычным условием перед `IsHierChild()` на
    сборке 906 выполнилась успешно:

    **XQuery:**

    ```xquery
    for $elem in subdivisions
    where $elem/is_disbanded = false()
      and IsHierChild($elem/id, 6327975429225669221)
    order by $elem/Hier()
    return $elem/Fields('id', 'name')
    ```

    Для этого варианта provider сформировал иерархический CTE и сохранил
    условие `is_disbanded = false()` во внешнем `WHERE`; `tools.xquery()`
    вернул строки. Результат относится только к точной многострочной записи на
    сборке 906. Для одинакового поведения на проверенных сборках 434 и 906
    используйте однострочную форму с `IsHierChild()` первым.

### IsHierChild()

**XQuery:**

```xquery
for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
```

Перед выполнением `tools.xquery()` удаляет условие `IsHierChild()` и заменяет
`/Hier()` на `/Hier(6327975429225669221, '-')`.

**SQL:**

```sql
WITH [subdivisions_cte]([id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],[__hlevel],[__sort_level],[__hcc]) AS
(
SELECT [id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],0 as [__hlevel],
cast((CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select top 1 1 from dbo.[subdivisions] f where f.parent_object_id = e.id) as [__hcc]
    FROM dbo.[subdivisions] e
    WHERE e.parent_object_id = @p0
 UNION ALL
SELECT e.[id],e.[code],e.[name],e.[org_id],e.[parent_object_id],e.[is_disbanded],e.[knowledge_parts],e.[tags],e.[experts],e.[place_id],e.[region_id],e.[kpi_profile_id],e.[kpi_profiles_id],e.[bonus_profile_id],e.[cost_center_id],e.[is_faculty],e.[modification_date],e.[app_instance_id],[__hlevel]+1,
cast((d.[__sort_level]+'.'+CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select 1 WHERE EXISTS (SELECT id FROM dbo.[subdivisions] f WHERE e.id = f.parent_object_id)) as [__hcc]
FROM dbo.[subdivisions] e
        INNER JOIN [subdivisions_cte] d
        ON e.parent_object_id = d.id    )
select t_elem.[id],t_elem.[name],[__hcc],[__hlevel] from [subdivisions_cte] t_elem order by t_elem.[__sort_level] asc
```

Параметры:

- `@p0 BigInt = 6327975429225669221`;
- `@p1 VarChar = -`.

Знак `-` означает выбор дочерних элементов без самого узла
`6327975429225669221`. Параметр `@p1` управляет режимом `Hier()` и не
подставляется в показанный SQL.

SQL дополнительно возвращает служебные поля `__hcc` и `__hlevel`, а порядок
иерархии хранится в `__sort_level`.

### IsHierChildOrSelf()

**XQuery:**

```xquery
for $elem in subdivisions where IsHierChildOrSelf($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
```

Перед выполнением `tools.xquery()` удаляет условие `IsHierChildOrSelf()` и
заменяет `/Hier()` на `/Hier(6327975429225669221, '+')`.

**SQL:**

```sql
WITH [subdivisions_cte]([id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],[__hlevel],[__sort_level],[__hcc]) AS
(
SELECT [id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],0 as [__hlevel],
cast((CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select top 1 1 from dbo.[subdivisions] f where f.parent_object_id = e.id) as [__hcc]
    FROM dbo.[subdivisions] e
    WHERE e.id = @p0
 UNION ALL
SELECT e.[id],e.[code],e.[name],e.[org_id],e.[parent_object_id],e.[is_disbanded],e.[knowledge_parts],e.[tags],e.[experts],e.[place_id],e.[region_id],e.[kpi_profile_id],e.[kpi_profiles_id],e.[bonus_profile_id],e.[cost_center_id],e.[is_faculty],e.[modification_date],e.[app_instance_id],[__hlevel]+1,
cast((d.[__sort_level]+'.'+CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select 1 WHERE EXISTS (SELECT id FROM dbo.[subdivisions] f WHERE e.id = f.parent_object_id)) as [__hcc]
FROM dbo.[subdivisions] e
        INNER JOIN [subdivisions_cte] d
        ON e.parent_object_id = d.id    )
select t_elem.[id],t_elem.[name],[__hcc],[__hlevel] from [subdivisions_cte] t_elem order by t_elem.[__sort_level] asc
```

Параметры:

- `@p0 BigInt = 6327975429225669221`;
- `@p1 VarChar = +`.

Знак `+` означает выбор дочерних элементов вместе с самим узлом
`6327975429225669221`. Параметр `@p1` управляет режимом `Hier()` и не
подставляется в показанный SQL.

### CatalogHierSubset()

**XQuery:**

```xquery
CatalogHierSubset('subdivisions', 6327975429225669221)
```

**SQL:**

```sql
WITH [subdivisions_cte]([id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],[__hlevel],[__sort_level],[__hcc]) AS
(
    SELECT [id],[code],[name],[org_id],[parent_object_id],[is_disbanded],[knowledge_parts],[tags],[experts],[place_id],[region_id],[kpi_profile_id],[kpi_profiles_id],[bonus_profile_id],[cost_center_id],[is_faculty],[modification_date],[app_instance_id],0 as [__hlevel],cast((CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select top 1 1 from dbo.subdivisions f where f.parent_object_id = e.id) as [__hcc]
    FROM dbo.subdivisions e
    WHERE parent_object_id = @p1
    UNION ALL
    SELECT e.[id],e.[code],e.[name],e.[org_id],e.[parent_object_id],e.[is_disbanded],e.[knowledge_parts],e.[tags],e.[experts],e.[place_id],e.[region_id],e.[kpi_profile_id],e.[kpi_profiles_id],e.[bonus_profile_id],e.[cost_center_id],e.[is_faculty],e.[modification_date],e.[app_instance_id],[__hlevel]+1, cast((d.[__sort_level]+'.'+
CAST(FLOOR(LOG10(ROW_NUMBER() over(order by e.id))) as varchar)+
cast(ROW_NUMBER() over(order by e.id) as varchar(256))) as varchar(256)) as [__sort_level],
(select 1 WHERE EXISTS (SELECT id FROM dbo.subdivisions f WHERE e.id = f.parent_object_id)) as [__hcc]

    FROM dbo.subdivisions e
        INNER JOIN [subdivisions_cte] d
        ON e.parent_object_id = d.id
)
select t_x.* from [subdivisions_cte] t_x
```

Из двух параметров в SQL используется только
`@p1 BigInt = 6327975429225669221`.
Параметр `@p0 VarChar = subdivisions` в команду не подставляется.

Для нового кода документация рекомендует `tools.xquery()` с
`IsHierChild()`. `CatalogHierSubset()` приведён как наблюдаемое поведение
сборки 906, а не как рекомендуемый вариант.

## Уникальные значения и сортировка

### distinct()

**XQuery:**

```xquery
for $elem in collaborators
return distinct($elem/fullname)
```

**SQL:**

```sql
select distinct(t_elem.[fullname])
from dbo.[collaborators] t_elem
```

`distinct()` превращается в `SELECT DISTINCT` для возвращаемого поля.

### order by

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
order by $elem/fullname
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[code] in (@p0,@p1)
order by t_elem.[fullname] asc
```

Параметры: `@p0 VarChar = 13744`, `@p1 VarChar = 109295`.

Фактический порядок:

```text
Анисимов Геннадий Федорович
Вилкова Ольга Николаевна
```

Ключевое слово `descending` меняет направление на `DESC`:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
order by $elem/fullname descending
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select t_elem.[id],t_elem.[fullname]
from dbo.[collaborators] t_elem
where t_elem.[code] in (@p0,@p1)
order by t_elem.[fullname] desc
```

Параметры: `@p0 VarChar = 13744`, `@p1 VarChar = 109295`.

Фактический порядок:

```text
Вилкова Ольга Николаевна
Анисимов Геннадий Федорович
```

Без явного `order by` provider этой сборки не добавлял сортировку по
`t_elem.id`. Поэтому утверждение, что такая сортировка появляется всегда, для
сборки 906 неверно.
