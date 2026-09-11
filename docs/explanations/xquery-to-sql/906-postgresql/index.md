# Как XQuery превращается в SQL: WebSoft HCM 906 и PostgreSQL

Примеры получены на WebSoft HCM Server `2023.2.906` с PostgreSQL provider
`1.24.4.18` и PostgreSQL `16.3`.

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
55 доступных полей плоского каталога. Даты преобразуются в строки через
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
	t_elem."consent_kedo",
	to_char(t_elem."consent_kedo_date", 'YYYY-MM-DD"T"HH24:MI:SS') "consent_kedo_date",
	t_elem."provider_legal_id",
	t_elem."snils",
	t_elem."cost_center_id",
	t_elem."disp_birthdate",
	t_elem."disp_birthdate_year",
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

Обе формы псевдонимов формируют корректный SQL для PostgreSQL:

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
	t_elem."id" "collaborator_id",
	t_elem."fullname" "collaborator_name"
from
	dbo."collaborators" t_elem
```

В версии provider `1.24.4.18` между именем поля и псевдонимом появился пробел.
Второй идентификатор PostgreSQL воспринимает как псевдоним; ключевое слово
`AS` необязательно. Запись псевдонимов внутри `Fields()` формирует такой же
SQL. Это отличие от provider `1.22.6.8` сборки 434, который склеивал два
идентификатора без пробела и создавал невалидную команду.

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
	end)= true
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
	end)= false)
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
	and t_elem."hire_date"<@p1
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
	inner join dbo."positions" t_pos on
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
	inner join dbo."positions" t_pos on
		t_elem."position_id" = t_pos."id"
	inner join dbo."appointment_types" t_app on
		t_pos."position_appointment_type_id" = t_app."id"
	where
		t_app."code" = @p0)
```

Параметр: `@p0 varchar = main`.

### Операторы join, ljoin и rjoin

Provider PostgreSQL `1.24.4.18` поддерживает явные операторы соединения. Это
отличие от версии `1.22.6.8` в сборке 434, где следующие запросы завершались
ошибкой до формирования SQL.

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
select
	t_elem."id",
	t_pos."name"
from
	dbo."collaborators" t_elem
inner join dbo."positions" t_pos on
	t_elem."position_id" = t_pos."id"
where
	t_elem."id" = @p0
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
select
	t_elem."id",
	t_pos."name"
from
	dbo."collaborators" t_elem
left join dbo."positions" t_pos on
	t_elem."position_id" = t_pos."id"
where
	t_elem."id" = @p0
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
select
	t_elem."id",
	t_pos."name"
from
	dbo."collaborators" t_elem
right join dbo."positions" t_pos on
	t_elem."position_id" = t_pos."id"
where
	t_elem."id" = @p0
```

Во всех трёх случаях параметр:
`@p0 bigint = 1105387902724063510`.

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
select
	t_elem."id"
from
	dbo."collaborators" t_elem
left join dbo."collaborators" t_excluded on
	t_elem."id" = t_excluded."id"
	and (t_excluded."code" in (@p0, @p1))
where
	t_excluded."id" is null
```

Параметры: `@p0 varchar = 13744`, `@p1 varchar = 109295`.

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
	(((t_elem."code" in (@p0, @p1))= false)
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
left join dbo."positions" "f527481313" on
	t_elem."position_id" = "f527481313"."id"
where
	"f527481313"."name" = @p0
```

Параметр: `@p0 varchar = Бизнес-тренер`.

`ForeignElem()` формирует `LEFT JOIN`. Условие по полю связанной таблицы
остаётся в `WHERE`, поэтому в этом запросе строки без связанной должности всё
равно отбрасываются. Provider `1.22.6.8` сборки 434 использовал `INNER JOIN`.

Число в служебном псевдониме может меняться между запусками, в том числе быть
отрицательным, поэтому на него нельзя ссылаться в прикладном коде.

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
`ForeignDispName($elem/position_id)` преобразуется в вызов
`foreigndispname(bigint)`, но такой функции нет. Для получения имени
связанного объекта используйте `ForeignElem()` или соединение каталогов.

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
select
	t_elem."id",
	t_elem."name"
from
	dbo."subdivisions" t_elem
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
    лишний `and`.

    **XQuery:**

    ```xquery
    for $elem in subdivisions where $elem/is_disbanded = false() and IsHierChild($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
    ```

    После переноса иерархии получается невалидный запрос:

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

    Для этой точной формы provider сформировал иерархический CTE и сохранил
    условие `is_disbanded = false()` во внешнем `WHERE`; `tools.xquery()`
    вернул строки. Для одинакового поведения на проверенных сборках 434 и 906
    используйте однострочную форму с `IsHierChild()` первым условием.

### IsHierChild()

**XQuery:**

```xquery
for $elem in subdivisions where IsHierChild($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
```

Перед выполнением `tools.xquery()` удаляет условие `IsHierChild()` и заменяет
`/Hier()` на `/Hier(6327975429225669221, '-')`.

**SQL:**

```sql
with recursive "subdivisions_cte"("id",
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
"kpi_profiles_id",
"bonus_profile_id",
"cost_center_id",
"is_faculty",
"modification_date",
"app_instance_id",
"__hlevel",
"__sort_level",
"__hcc") as
(
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
	"kpi_profiles_id",
	"bonus_profile_id",
	"cost_center_id",
	"is_faculty",
	"modification_date",
	"app_instance_id",
	0 as __hlevel,
	cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
	e."kpi_profiles_id",
	e."bonus_profile_id",
	e."cost_center_id",
	e."is_faculty",
	e."modification_date",
	e."app_instance_id",
	"__hlevel" + 1,
	cast((d."__sort_level" || '.' || cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
inner join "subdivisions_cte" d
        on
	e.parent_object_id = d.id )
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

### IsHierChildOrSelf()

**XQuery:**

```xquery
for $elem in subdivisions where IsHierChildOrSelf($elem/id, 6327975429225669221) order by $elem/Hier() return $elem/Fields('id', 'name')
```

SQL отличается от `IsHierChild()` начальным условием `e."id" = @p0` вместо
`e."parent_object_id" = @p0`:

**SQL:**

```sql
with recursive "subdivisions_cte"("id",
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
"kpi_profiles_id",
"bonus_profile_id",
"cost_center_id",
"is_faculty",
"modification_date",
"app_instance_id",
"__hlevel",
"__sort_level",
"__hcc") as
(
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
	"kpi_profiles_id",
	"bonus_profile_id",
	"cost_center_id",
	"is_faculty",
	"modification_date",
	"app_instance_id",
	0 as __hlevel,
	cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
	e."kpi_profiles_id",
	e."bonus_profile_id",
	e."cost_center_id",
	e."is_faculty",
	e."modification_date",
	e."app_instance_id",
	"__hlevel" + 1,
	cast((d."__sort_level" || '.' || cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
inner join "subdivisions_cte" d
        on
	e.parent_object_id = d.id )
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

### CatalogHierSubset()

**XQuery:**

```xquery
CatalogHierSubset('subdivisions', 6327975429225669221)
```

Provider также строит рекурсивный CTE. Начальная ветвь выбирает дочерние
элементы:

**SQL:**

```sql
with recursive "subdivisions_cte"("id",
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
"kpi_profiles_id",
"bonus_profile_id",
"cost_center_id",
"is_faculty",
"modification_date",
"app_instance_id",
"__hlevel",
"__sort_level",
"__hcc") as
(
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
	"kpi_profiles_id",
	"bonus_profile_id",
	"cost_center_id",
	"is_faculty",
	"modification_date",
	"app_instance_id",
	0 as "__hlevel",
	cast((cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
	e."kpi_profiles_id",
	e."bonus_profile_id",
	e."cost_center_id",
	e."is_faculty",
	e."modification_date",
	e."app_instance_id",
	"__hlevel" + 1,
	cast((d."__sort_level" || '.' ||
cast(FLOOR(LOG(row_number() over(order by e."id"))) as varchar)||
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
inner join "subdivisions_cte" d
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

Явные `nulls first` для возрастания и `nulls last` для убывания отличаются от
стандартного порядка `NULL` в PostgreSQL и сохраняют поведение, привычное для
MSSQL.
