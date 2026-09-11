# Как XQuery превращается в SQL: WebSoft HCM 434 и PostgreSQL

Примеры получены на WebSoft HCM Server `2022.1.3.434` с PostgreSQL provider
`1.22.6.8` и PostgreSQL `16.3`.

`LuceneFTIndex` и `XQueryTemplateCache` были отключены.

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
  t_elem."id",
  t_elem."fullname",
  t_elem."position_name"
from
  dbo."collaborators" t_elem
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
48 доступных полей плоского каталога. Даты преобразуются в строки через
`to_char()`:

**SQL:**

```sql
select
  t_elem."id",
  t_elem."code",
  t_elem."fullname",
  t_elem."login",
  t_elem."short_login",
  t_elem."lowercase_login",
  t_elem."email",
  t_elem."phone",
  t_elem."mobile_phone",
  to_char(t_elem."birth_date", 'YYYY-MM-DD"T"HH24:MI:SS') "birth_date",
  t_elem."sex",
  t_elem."pict_url",
  t_elem."position_id",
  t_elem."position_name",
  t_elem."position_parent_id",
  t_elem."position_parent_name",
  t_elem."org_id",
  t_elem."org_name",
  t_elem."place_id",
  t_elem."region_id",
  t_elem."category_id",
  t_elem."web_banned",
  t_elem."is_arm_admin",
  t_elem."is_content_admin",
  t_elem."is_application_admin",
  t_elem."role_id",
  t_elem."is_candidate",
  t_elem."candidate_status_type_id",
  t_elem."candidate_id",
  t_elem."is_outstaff",
  t_elem."is_dismiss",
  to_char(t_elem."position_date", 'YYYY-MM-DD"T"HH24:MI:SS') "position_date",
  to_char(t_elem."hire_date", 'YYYY-MM-DD"T"HH24:MI:SS') "hire_date",
  to_char(t_elem."dismiss_date", 'YYYY-MM-DD"T"HH24:MI:SS') "dismiss_date",
  t_elem."in_request_black_list",
  t_elem."allow_personal_chat_request",
  t_elem."level_id",
  t_elem."grade_id",
  t_elem."knowledge_parts",
  t_elem."tags",
  t_elem."experts",
  t_elem."person_object_profile_id",
  t_elem."current_state",
  to_char(t_elem."next_state_date", 'YYYY-MM-DD"T"HH24:MI:SS') "next_state_date",
  t_elem."development_potential_id",
  t_elem."efficiency_estimation_id",
  to_char(t_elem."modification_date", 'YYYY-MM-DD"T"HH24:MI:SS') "modification_date",
  t_elem."app_instance_id"
from
  dbo."collaborators" t_elem
where
  t_elem."id" = @p0
```

Параметр: `@p0 bigint = 1105387902724063510`. Состав `SELECT` зависит от
схемы каталога конкретной сборки.

### Псевдонимы полей

Обе формы псевдонимов формируют невалидный PostgreSQL:

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
  t_elem."id""collaborator_id",
  t_elem."fullname""collaborator_name"
from
  dbo."collaborators" t_elem
```

Между именем поля и псевдонимом нет ни пробела, ни `AS`, поэтому PostgreSQL
воспринимает всю конструкцию как имя столбца. Запись псевдонимов внутри
`Fields()` приводит к тому же невалидному SQL. В MSSQL аналогичная склейка
двух идентификаторов допустима, но для PostgreSQL этот синтаксис непереносим.

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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  t_elem."id" = @p0
```

Параметр: `@p0 bigint = 1105387902724063510`.

Остальные операторы сравнения также параметризуются:

| Условие XQuery | Условие SQL |
| --- | --- |
| `$elem/id != 1105387902724063510` | `((t_elem."id"<>@p0) OR ((t_elem."id")  IS NULL))` |
| `$elem/id > 1105387902724063510` | `t_elem."id">@p0` |
| `$elem/id >= 1105387902724063510` | `t_elem."id">=@p0` |
| `$elem/id < 1105387902724063510` | `t_elem."id"<@p0` |
| `$elem/id <= 1105387902724063510` | `t_elem."id"<=@p0` |

Во всех пяти запросах параметр имеет тип `bigint`. Для `!=` provider
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
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  t_elem."code" = @p0
```

Параметр: `@p0 varchar = 13744`.

### Поиск подстроки через contains()

**XQuery:**

```xquery
for $elem in collaborators
where contains($elem/fullname, 'Анисимов')
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  (t_elem."fullname" ilike @p0)
```

Параметр уже содержит маску: `@p0 varchar = %Анисимов%`. Provider использует
регистронезависимый `ILIKE`, а не `LIKE`.

Специальные символы обрабатываются неодинаково:

| Аргумент `contains()` | Значение `@p0` | Результат |
| --- | --- | --- |
| `%` | `%%%` | `%` остаётся SQL-маской |
| `_` | `%\_%` | `_` экранируется обратной косой чертой |
| `[` | `%[%` | `[` ищется как обычный символ |

Для `%` сохраняется значение SQL-маски. Перед `_` provider добавляет обратную
косую черту, поэтому PostgreSQL ищет подчёркивание как литерал, а не как маску
одного символа.

Отрицание включает строки с `NULL`:

**XQuery:**

```xquery
for $elem in collaborators
where contains($elem/fullname, 'Анисимов') = false()
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  (not(t_elem."fullname" ilike @p0)
  or t_elem."fullname" is null)
```

Параметр: `@p0 varchar = %Анисимов%`.

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
select
  t_elem."id",
  t_elem."is_dismiss"
from
  dbo."collaborators" t_elem
where
  ((t_elem."is_dismiss" = false)
  or ((t_elem."is_dismiss") is null))
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
select
  t_elem."id",
  t_elem."is_dismiss"
from
  dbo."collaborators" t_elem
where
  t_elem."is_dismiss" <> false
```

Равенство `false()` включает `NULL`, а отрицание `false()` — только записи со
значением `true`.

#### true()

**XQuery:**

```xquery
for $elem in collaborators
where $elem/is_dismiss = true()
return $elem/Fields('id', 'is_dismiss')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."is_dismiss"
from
  dbo."collaborators" t_elem
where
  t_elem."is_dismiss" = true
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
select
  t_elem."id",
  t_elem."is_dismiss"
from
  dbo."collaborators" t_elem
where
  ((t_elem."is_dismiss" <> true)
  or ((t_elem."is_dismiss") is null))
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
select
  t_elem."id",
  t_elem."position_id"
from
  dbo."collaborators" t_elem
where
  t_elem."position_id" is null
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
select
  t_elem."id",
  t_elem."position_id"
from
  dbo."collaborators" t_elem
where
  t_elem."position_id" is not null
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
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  t_elem."code" is null
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
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  t_elem."code" is not null
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
select
  t_elem."id",
  t_elem."birth_date"
from
  dbo."collaborators" t_elem
where
  (case
  when t_elem."birth_date" is null then true
  else false
  end) = true
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
select
  t_elem."id",
  t_elem."birth_date"
from
  dbo."collaborators" t_elem
where
  (((case
      when t_elem."birth_date" is null then true
      else false
      end) = false)
  or (((case
        when t_elem."birth_date" is null then true
        else false
        end)) is null))
```

Форма `IsEmpty($elem/birth_date) != true()` заменяет `= false` на `<> true`;
остальная конструкция SQL не меняется.

### Дата

**XQuery:**

```xquery
for $elem in collaborators
where $elem/hire_date = date('2010-03-17')
return $elem/Fields('id', 'hire_date')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."hire_date"
from
  dbo."collaborators" t_elem
where
  t_elem."hire_date" = @p0
```

Параметр: `@p0 timestamp = 2010-03-17T00:00:00`.

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
select
  t_elem."id",
  t_elem."hire_date"
from
  dbo."collaborators" t_elem
where
  t_elem."hire_date" >= @p0
  and t_elem."hire_date" < @p1
```

Параметры: `@p0 timestamp = 2010-03-01T00:00:00`,
`@p1 timestamp = 2010-04-01T00:00:00`.

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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  (((t_elem."is_dismiss" = false)
    or ((t_elem."is_dismiss") is null))
  and (t_elem."fullname" ilike @p0))
  or t_elem."id" = @p1
```

Параметры:

- `@p0 varchar = %Анисимов%`;
- `@p1 bigint = 1105387902724063521`.

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
  t_elem."id",
  t_elem."fullname",
  t_pos."name"
from
  dbo."positions" t_pos,
  dbo."collaborators" t_elem
where
  t_elem."position_id" = t_pos."id"
  and t_elem."id" = @p0
```

Параметр: `@p0 bigint = 1105387902724063510`.

Связь каталогов остаётся в `WHERE`; отдельный `JOIN` не создаётся.

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
select
  t_elem."id",
  t_elem."position_name"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      t_elem."id"
    from
      dbo."collaborators" t_elem
    inner join
      dbo."positions" t_pos
    on
      t_elem."position_id" = t_pos."id"
    where
      t_pos."name" = @p0)
```

Параметр: `@p0 varchar = Руководитель отдела обучения и развития`.

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
select
  t_elem."id",
  t_elem."position_name"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      t_elem."id"
    from
      dbo."collaborators" t_elem
    inner join
      dbo."positions" t_pos
    on
      t_elem."position_id" = t_pos."id"
    inner join
      dbo."appointment_types" t_app
    on
      t_pos."position_appointment_type_id" = t_app."id"
    where
      t_app."code" = @p0)
```

Параметр: `@p0 varchar = main`.

### Неподдерживаемые join, ljoin и rjoin

Все четыре запроса этого подраздела завершаются ошибкой до формирования SQL.

#### join

**XQuery:**

```xquery
for $pos in positions join $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

#### ljoin

**XQuery:**

```xquery
for $pos in positions ljoin $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

#### rjoin

**XQuery:**

```xquery
for $pos in positions rjoin $elem in collaborators
  on $elem/position_id = $pos/id
where $elem/id = 1105387902724063510
return $elem/Fields('id') $pos/Fields('name')
```

#### Неподдерживаемое антисоединение через ljoin и MatchSome()

**XQuery:**

```xquery
for $excluded in collaborators ljoin $elem in collaborators
  on $elem/id = $excluded/id
    and MatchSome($excluded/code, ('13744', '109295'))
where $excluded/id = null()
return $elem/Fields('id')
```

Первые три запроса не разбираются из-за явных операторов `join`, `ljoin` и
`rjoin`. Антисоединение содержит тот же неподдерживаемый `ljoin`, поэтому до
обработки `MatchSome()` дело не доходит. Рабочие варианты — перечисление
каталогов через запятую без псевдонимов, `some/satisfies` или `ForeignElem()`.

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
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  (t_elem."code" in (@p0, @p1))
```

Параметры: `@p0 varchar = 13744`, `@p1 varchar = 109295`.

Для множественного поля provider использует оператор пересечения массивов
`&&`:

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
select
  t_elem."id",
  t_elem."fullname",
  t_elem."category_id"
from
  dbo."collaborators" t_elem
where
  ((t_elem."category_id" && @p0))
```

Параметр: `p0 varchar[] = ["123","456"]`.

При прямой записи SQL параметру соответствует
`ARRAY['123', '456']::varchar[]`. Тип массива определяется типом поля; для
`category_id` это `varchar[]`.

Отрицание через `false()` в PostgreSQL работает:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295')) = false()
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  (((t_elem."code" in (@p0, @p1)) = false)
  or (((t_elem."code" in (@p0, @p1))) is null))
```

Параметры: `@p0 varchar = 13744`, `@p1 varchar = 109295`. В отличие от
MSSQL, PostgreSQL разрешает сравнивать булев результат `IN` с `false`.

Две формы с `not` не работают:

**XQuery:**

```xquery
for $elem in collaborators
where not MatchSome($elem/code, ('13744', '109295'))
return $elem/Fields('id', 'code')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  "not"(t_elem."code" in (@p0, @p1))
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
select
  t_elem."id",
  t_elem."code"
from
  dbo."collaborators" t_elem
where
  not()
```

Первая команда пытается вызвать функцию `not(boolean)`, второй SQL содержит
пустой вызов. Для отрицания нужно использовать сравнение
`MatchSome(...) = false()`.

## ForeignElem()

**XQuery:**

```xquery
for $elem in collaborators
where ForeignElem($elem/position_id)/name = 'Бизнес-тренер'
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."position_id"
from
  dbo."collaborators" t_elem
inner join
  dbo."positions" "f866364379"
on
  t_elem."position_id" = "f866364379"."id"
where
  "f866364379"."name" = @p0
```

Параметр: `@p0 varchar = Бизнес-тренер`.

`ForeignElem()` стал обычным `INNER JOIN`. Число в служебном псевдониме
может меняться между запусками, в том числе быть отрицательным, поэтому на
него нельзя ссылаться в прикладном коде.

## ForeignDispName()

Документированная форма вызова как атрибута не работает в условии:

**XQuery:**

```xquery
for $elem in collaborators
where $elem/position_id/ForeignDispName = 'Бизнес-тренер'
return $elem/Fields('id', 'position_id')
```

В условии PostgreSQL не находит столбец `ForeignDispName`. В сортировке
получается невалидный SQL:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
order by $elem/position_id/ForeignDispName
return $elem/Fields('id', 'position_id')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."position_id"
from
  dbo."collaborators" t_elem
where
  (t_elem."code" in (@p0, @p1))
order by
  t_elem."position_id"."ForeignDispName" asc nulls first
```

PostgreSQL воспринимает `position_id` как имя таблицы. Функциональная форма
`ForeignDispName($elem/position_id)` оставляет в SQL вызов
`foreigndispname(bigint)`, но такой функции нет. Для
получения имени связанного объекта используйте `ForeignElem()` или соединение
каталогов.

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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      "id"
    from
      dbo."collaborator"
    where
      xmlexists('/collaborator/*[contains(text(),"Анисимов")]' passing by ref "data") = true)
```

При `LuceneFTIndex=False` PostgreSQL использует `xmlexists()`, а не
полнотекстовый индекс. Внешняя выборка идёт из плоского каталога
`dbo."collaborators"`, а поиск — из XML-колонки `data` документной таблицы
`dbo."collaborator"`.

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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      "id"
    from
      dbo."collaborator"
    where
      xmlexists('/collaborator/custom_elems/custom_elem[name = "f_zsbu" and value = "зеленый"]' passing by ref "data"))
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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      "id"
    from
      dbo."collaborator"
    where
      xmlexists('/collaborator/custom_elems/custom_elem[name = "f_zsbu" and contains(value,  "зелен")]' passing by ref "data"))
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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  t_elem."id" in (
    select
      "id"
    from
      dbo."collaborator"
    where
      xmlexists('/collaborator/custom_elems/custom_elem[name = "is_universal" and value = "true"]' passing by ref "data"))
```

В PostgreSQL булево значение превращается в строковое сравнение
`value = "true"`; приведение через `xs:boolean()` не используется.

## Иерархия

`IsHierChild()` и `IsHierChildOrSelf()` — псевдофункции, а не обычные условия
SQL provider. В прикладном коде используйте их только через `tools.xquery()`:
функция предварительно удаляет иерархическое условие и переносит ID базового
узла и режим выборки в `$elem/Hier()`. Прямая передача исходной строки
provider обходит эту обработку.

### Ограничения

#### Обязательный order by Hier()

`IsHierChild()` нельзя использовать без `order by $elem/Hier()`.

Для однострочного XQuery:

```xquery
for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) return $elem/Fields('id', 'name')
```

`tools.xquery()` удаляет условие целиком:

```xquery
for $elem in subdivisions return $elem/Fields('id', 'name')
```

Для многострочного XQuery:

```xquery
for $elem in subdivisions
where IsHierChild($elem/id, 6327975429225669221)
return $elem/Fields('id', 'name')
```

после предварительной обработки `tools.xquery()` остаётся пустой `where`:

```xquery
for $elem in subdivisions
where
return $elem/Fields('id', 'name')
```

В обоих случаях provider всё же формирует одинаковый SQL без `WHERE`:

```sql
select
  t_elem."id",
  t_elem."name"
from
  dbo."subdivisions" t_elem
```

Этот SQL синтаксически корректен, но логически не соответствует исходному
запросу: иерархический фильтр потерян. На стенде контрольный запрос и обе
формы с `IsHierChild()` вернули один полный каталог из 19 ID. Многострочный
`IsHierChildOrSelf()` без `Hier()` дал тот же результат.

!!! danger "Опасность"
    Наличие пустого `where` в обработанном XQuery не гарантирует ошибку.
    Provider проигнорировал его и выполнил выборку всего каталога. Переносы
    строк не компенсируют отсутствие `order by $elem/Hier()`.

#### Дополнительные условия

В многострочном запросе `IsHierChild()` и `IsHierChildOrSelf()` могут стоять
до или после обычного условия. Если псевдофункция стоит первой:

```xquery
for $elem in subdivisions
where IsHierChild($elem/id, 6327975429225669221)
  and $elem/is_disbanded = false()
order by $elem/Hier()
return $elem/Fields('id', 'name')
```

`tools.xquery()` формирует:

```xquery
for $elem in subdivisions
where
 and $elem/is_disbanded = false()
order by $elem/Hier(  6327975429225669221,'-')
return $elem/Fields('id', 'name')
```

При обратном порядке:

```xquery
for $elem in subdivisions
where $elem/is_disbanded = false()
  and IsHierChild($elem/id, 6327975429225669221)
order by $elem/Hier()
return $elem/Fields('id', 'name')
```

в обработанном XQuery остаётся `and` в конце строки:

```xquery
for $elem in subdivisions
where $elem/is_disbanded = false()
 and
order by $elem/Hier(  6327975429225669221,'-')
return $elem/Fields('id', 'name')
```

Обе обработанные формы содержат остаточный `and`: в первой он стоит перед
обычным условием, во второй — после него. Несмотря на это, provider
сформировал одинаковый SQL, а `tools.xquery()` вернул одни и те же шесть ID.
Это подтверждённое поведение сборки 434. Порядок с псевдофункцией первой
используется далее как единообразный вариант, поскольку он работает также в
однострочной записи.

В однострочном запросе порядок уже влияет на результат. Рабочая форма:

```xquery
for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) and $elem/is_disbanded = false() order by $elem/Hier() return $elem/Fields('id', 'name')
```

после обработки становится корректным XQuery:

```xquery
for $elem in subdivisions where $elem/is_disbanded = false() order by $elem/Hier(  6327975429225669221,'-') return $elem/Fields('id', 'name')
```

Если переставить условия:

```xquery
for $elem in subdivisions where $elem/is_disbanded = false() and IsHierChild($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
```

препроцессор удаляет `where`, но оставляет первое условие и `and`:

```xquery
for $elem in subdivisions $elem/is_disbanded = false() and order by $elem/Hier(  6327975429225669221,'-') return $elem/Fields('id', 'name')
```

Provider интерпретирует `and` как имя каталога и завершает запрос ошибкой:

```text
Npgsql.PostgresException
42P01: relation "dbo.and" does not exist

POSITION: 15
```

!!! warning "Предупреждение"
    В многострочной записи проверены оба порядка условий, и после обработки
    оба оставляют `and`. В однострочной записи работает только порядок с
    `IsHierChild()` перед дополнительным условием. Чтобы использовать один
    порядок в обоих форматах, ставьте псевдофункцию первой.

### Рабочие примеры

#### IsHierChild()

**XQuery:**

```xquery
for $elem in subdivisions
where IsHierChild($elem/id, 6327975429225669221)
order by $elem/Hier()
return $elem/Fields('id', 'name')
```

Перед выполнением `tools.xquery()` удаляет условие `IsHierChild()` и заменяет
`/Hier()` на `/Hier(6327975429225669221, '-')`.

Однострочная форма, многострочная форма с LF и многострочная форма с CRLF
вернули одни и те же шесть ID в одинаковом порядке.

**SQL:**

```sql
with recursive "subdivisions_cte"(
  "id",
  "code",
  "name",
  "org_id",
  "parent_object_id",
  "is_disbanded",
  "knowledge_parts",
  "tags",
  "experts",
  "place_id",
  "region_id",
  "kpi_profile_id",
  "bonus_profile_id",
  "cost_center_id",
  "is_faculty",
  "modification_date",
  "app_instance_id",
  "__hlevel",
  "__sort_level",
  "__hcc"
) as (
  select
    "id",
    "code",
    "name",
    "org_id",
    "parent_object_id",
    "is_disbanded",
    "knowledge_parts",
    "tags",
    "experts",
    "place_id",
    "region_id",
    "kpi_profile_id",
    "bonus_profile_id",
    "cost_center_id",
    "is_faculty",
    "modification_date",
    "app_instance_id",
    0 as __hlevel,
    cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      from
        dbo."subdivisions" f
      where
        f.parent_object_id = e.id
      limit 1) as "__hcc"
  from
    dbo."subdivisions" e
  where
    e.parent_object_id = @p0
  union all
  select
    e."id",
    e."code",
    e."name",
    e."org_id",
    e."parent_object_id",
    e."is_disbanded",
    e."knowledge_parts",
    e."tags",
    e."experts",
    e."place_id",
    e."region_id",
    e."kpi_profile_id",
    e."bonus_profile_id",
    e."cost_center_id",
    e."is_faculty",
    e."modification_date",
    e."app_instance_id",
    "__hlevel" + 1,
    cast((d."__sort_level" || '.' || cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      where
        exists (
          select
            id
          from
            dbo."subdivisions" f
          where
            e.id = f.parent_object_id)) as "__hcc"
  from
    dbo."subdivisions" e
  inner join
    "subdivisions_cte" d
  on
    e.parent_object_id = d.id
)
select
  t_elem."id",
  t_elem."name",
  "__hcc",
  "__hlevel"
from
  "subdivisions_cte" t_elem
order by
  t_elem."__sort_level" asc nulls first
```

Параметры:

- `@p0 bigint = 6327975429225669221`;
- `@p1 varchar = -`.

Знак `-` означает выбор дочерних элементов без самого исходного узла.
Параметр `@p1` управляет режимом `Hier()` и не подставляется в показанный SQL.

PostgreSQL-вариант использует `WITH RECURSIVE`, конкатенацию через `||` и
`limit 1`. SQL дополнительно возвращает служебные поля `__hcc` и `__hlevel`,
а порядок иерархии хранится в `__sort_level`.

#### IsHierChildOrSelf()

**XQuery:**

```xquery
for $elem in subdivisions
where IsHierChildOrSelf($elem/id, 6327975429225669221)
order by $elem/Hier()
return $elem/Fields('id', 'name')
```

SQL отличается от `IsHierChild()` начальным условием `e."id" = @p0` вместо
`e."parent_object_id" = @p0`:

**SQL:**

```sql
with recursive "subdivisions_cte"(
  "id",
  "code",
  "name",
  "org_id",
  "parent_object_id",
  "is_disbanded",
  "knowledge_parts",
  "tags",
  "experts",
  "place_id",
  "region_id",
  "kpi_profile_id",
  "bonus_profile_id",
  "cost_center_id",
  "is_faculty",
  "modification_date",
  "app_instance_id",
  "__hlevel",
  "__sort_level",
  "__hcc"
) as (
  select
    "id",
    "code",
    "name",
    "org_id",
    "parent_object_id",
    "is_disbanded",
    "knowledge_parts",
    "tags",
    "experts",
    "place_id",
    "region_id",
    "kpi_profile_id",
    "bonus_profile_id",
    "cost_center_id",
    "is_faculty",
    "modification_date",
    "app_instance_id",
    0 as __hlevel,
    cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      from
        dbo."subdivisions" f
      where
        f.parent_object_id = e.id
      limit 1) as "__hcc"
  from
    dbo."subdivisions" e
  where
    e.id = @p0
  union all
  select
    e."id",
    e."code",
    e."name",
    e."org_id",
    e."parent_object_id",
    e."is_disbanded",
    e."knowledge_parts",
    e."tags",
    e."experts",
    e."place_id",
    e."region_id",
    e."kpi_profile_id",
    e."bonus_profile_id",
    e."cost_center_id",
    e."is_faculty",
    e."modification_date",
    e."app_instance_id",
    "__hlevel" + 1,
    cast((d."__sort_level" || '.' || cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      where
        exists (
          select
            id
          from
            dbo."subdivisions" f
          where
            e.id = f.parent_object_id)) as "__hcc"
  from
    dbo."subdivisions" e
  inner join
    "subdivisions_cte" d
  on
    e.parent_object_id = d.id
)
select
  t_elem."id",
  t_elem."name",
  "__hcc",
  "__hlevel"
from
  "subdivisions_cte" t_elem
order by
  t_elem."__sort_level" asc nulls first
```

Перед выполнением `/Hier()` заменяется на
`/Hier(6327975429225669221, '+')`. Параметр `@p1 varchar = +` управляет
режимом и не подставляется в SQL.

Однострочная и многострочная формы вернули одни и те же семь ID в
одинаковом порядке; первым был базовый узел `6327975429225669221`.

#### IsHierChild() с дополнительным условием

**XQuery:**

```xquery
for $elem in subdivisions
where IsHierChild($elem/id, 6327975429225669221)
  and $elem/is_disbanded = false()
order by $elem/Hier()
return $elem/Fields('id', 'name')
```

`tools.xquery()` преобразует её в:

```xquery
for $elem in subdivisions
where
 and $elem/is_disbanded = false()
order by $elem/Hier(  6327975429225669221,'-')
return $elem/Fields('id', 'name')
```

Ведущий `and` в обработанном XQuery — подтверждённый результат
препроцессинга, а не опечатка. Provider принимает эту форму и формирует
корректный SQL.

**SQL:**

```sql
with recursive "subdivisions_cte"(
  "id",
  "code",
  "name",
  "org_id",
  "parent_object_id",
  "is_disbanded",
  "knowledge_parts",
  "tags",
  "experts",
  "place_id",
  "region_id",
  "kpi_profile_id",
  "bonus_profile_id",
  "cost_center_id",
  "is_faculty",
  "modification_date",
  "app_instance_id",
  "__hlevel",
  "__sort_level",
  "__hcc"
) as (
  select
    "id",
    "code",
    "name",
    "org_id",
    "parent_object_id",
    "is_disbanded",
    "knowledge_parts",
    "tags",
    "experts",
    "place_id",
    "region_id",
    "kpi_profile_id",
    "bonus_profile_id",
    "cost_center_id",
    "is_faculty",
    "modification_date",
    "app_instance_id",
    0 as __hlevel,
    cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      from
        dbo."subdivisions" f
      where
        f.parent_object_id = e.id
      limit 1) as "__hcc"
  from
    dbo."subdivisions" e
  where
    e.parent_object_id = @p0
  union all
  select
    e."id",
    e."code",
    e."name",
    e."org_id",
    e."parent_object_id",
    e."is_disbanded",
    e."knowledge_parts",
    e."tags",
    e."experts",
    e."place_id",
    e."region_id",
    e."kpi_profile_id",
    e."bonus_profile_id",
    e."cost_center_id",
    e."is_faculty",
    e."modification_date",
    e."app_instance_id",
    "__hlevel" + 1,
    cast((d."__sort_level" || '.' || cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      where
        exists (
          select
            id
          from
            dbo."subdivisions" f
          where
            e.id = f.parent_object_id)) as "__hcc"
  from
    dbo."subdivisions" e
  inner join
    "subdivisions_cte" d
  on
    e.parent_object_id = d.id
)
select
  t_elem."id",
  t_elem."name",
  "__hcc",
  "__hlevel"
from
  "subdivisions_cte" t_elem
where
  ((t_elem."is_disbanded" = false)
  or ((t_elem."is_disbanded") is null))
order by
  t_elem."__sort_level" asc nulls first
```

Provider использует тот же рекурсивный CTE, что и в примере
`IsHierChild()`, и добавляет проверку `is_disbanded = false()` во внешний
`WHERE`. На стенде запрос вернул те же шесть ID: все выбранные подразделения
имели `is_disbanded = false()`.

### CatalogHierSubset()

**XQuery:**

```xquery
CatalogHierSubset('subdivisions', 6327975429225669221)
```

Provider также строит рекурсивный CTE. Начальная ветвь выбирает дочерние
элементы:

**SQL:**

```sql
with recursive "subdivisions_cte"(
  "id",
  "code",
  "name",
  "org_id",
  "parent_object_id",
  "is_disbanded",
  "knowledge_parts",
  "tags",
  "experts",
  "place_id",
  "region_id",
  "kpi_profile_id",
  "bonus_profile_id",
  "cost_center_id",
  "is_faculty",
  "modification_date",
  "app_instance_id",
  "__hlevel",
  "__sort_level",
  "__hcc"
) as (
  select
    "id",
    "code",
    "name",
    "org_id",
    "parent_object_id",
    "is_disbanded",
    "knowledge_parts",
    "tags",
    "experts",
    "place_id",
    "region_id",
    "kpi_profile_id",
    "bonus_profile_id",
    "cost_center_id",
    "is_faculty",
    "modification_date",
    "app_instance_id",
    0 as "__hlevel",
    cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      from
        dbo.subdivisions f
      where
        f.parent_object_id = e.id
      limit 1) as "__hcc"
  from
    dbo.subdivisions e
  where
    parent_object_id = @p1
  union all
  select
    e."id",
    e."code",
    e."name",
    e."org_id",
    e."parent_object_id",
    e."is_disbanded",
    e."knowledge_parts",
    e."tags",
    e."experts",
    e."place_id",
    e."region_id",
    e."kpi_profile_id",
    e."bonus_profile_id",
    e."cost_center_id",
    e."is_faculty",
    e."modification_date",
    e."app_instance_id",
    "__hlevel" + 1,
    cast((d."__sort_level" || '.' ||
      cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar) ||
      cast(row_number() over(order by e."id") as varchar(256))) as varchar(256)) as "__sort_level",
    (
      select
        1
      where
        exists (
          select
            id
          from
            dbo.subdivisions f
          where
            e.id = f.parent_object_id)) as "__hcc"
  from
    dbo.subdivisions e
  inner join
    "subdivisions_cte" d
  on
    e.parent_object_id = d.id
)
select
  t_x.*
from
  "subdivisions_cte" t_x
```

Из двух параметров в SQL используется только
`@p1 bigint = 6327975429225669221`. Параметр
`@p0 varchar = subdivisions` в команду не подставляется.

Для нового кода документация рекомендует `tools.xquery()` с
`IsHierChild()`. `CatalogHierSubset()` приведён как наблюдаемое поведение
сборки 434, а не как рекомендуемый вариант.

При повторной проверке `CatalogHierSubset()` вернул тот же набор из шести ID,
что и `IsHierChild()`, но в другом порядке. В итоговом SQL нет `ORDER BY`,
поэтому на порядок строк полагаться нельзя.

## Уникальные значения и сортировка

### distinct()

**XQuery:**

```xquery
for $elem in collaborators
return distinct($elem/fullname)
```

**SQL:**

```sql
select
  distinct(t_elem."fullname")
from
  dbo."collaborators" t_elem
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
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  (t_elem."code" in (@p0, @p1))
order by
  t_elem."fullname" asc nulls first
```

Параметры: `@p0 varchar = 13744`, `@p1 varchar = 109295`.

Фактический порядок:

```text
Анисимов Геннадий Федорович
Вилкова Ольга Николаевна
```

Ключевое слово `descending` меняет направление и положение `NULL`:

**XQuery:**

```xquery
for $elem in collaborators
where MatchSome($elem/code, ('13744', '109295'))
order by $elem/fullname descending
return $elem/Fields('id', 'fullname')
```

**SQL:**

```sql
select
  t_elem."id",
  t_elem."fullname"
from
  dbo."collaborators" t_elem
where
  (t_elem."code" in (@p0, @p1))
order by
  t_elem."fullname" desc nulls last
```

Параметры: `@p0 varchar = 13744`, `@p1 varchar = 109295`.

Фактический порядок:

```text
Вилкова Ольга Николаевна
Анисимов Геннадий Федорович
```

Явные `nulls first` для возрастания и `nulls last` для убывания отличаются от
стандартного порядка `NULL` в PostgreSQL и сохраняют поведение, привычное для
MSSQL.
