# Как добавить или обновить элементы общего справочника

Рецепт добавляет несколько элементов в общий справочник через
`ms_tools.obtain_shared_list_elem()` и обновляет элемент, если его ключ уже
существует.

## Условия применения

- Реализация метода изучена в поставках WebSoft HCM Server `2022.1.3.434`,
  `2023.2.906` и `2025.1.1333`.
- Пример предназначен для серверного кода. Перенаправление вызова с клиента на
  сервер, предусмотренное методом, в рамках рецепта не проверялось.
- Перед массовым изменением сохраните резервную копию конфигурации.
- Используемая СУБД не имеет значения: метод изменяет XML-документ общего
  справочника.

## Решение

Функция принимает путь к списку, XML-шаблон одного элемента и массив данных.
Значения `id` и `name` присваиваются XML-полям, а не вставляются в XML-строку:

```js
function upsertSharedListItems(listPath, elementTemplate, items)
{
  var item;
  var element;

  for (item in items)
  {
    element = OpenDocFromStr(elementTemplate).TopElem;
    element.id = item.code;
    element.name = item.name;

    ms_tools.obtain_shared_list_elem(listPath, item.code, element);
  }
}

var personStates = [
  {code: 'parental_leave', name: 'Декрет'},
  {code: 'sick_leave', name: 'Больничный'},
  {code: 'vacation', name: 'Отпуск'}
];

var personStateTemplate =
  '<person_state ' +
  'SPXML-FORM="x-local://wtv/wtv_lists.xmd" ' +
  'SPXML-FORM-ELEM="lists.person_states.person_state">' +
  '<id></id><name></name>' +
  '</person_state>';

upsertSharedListItems(
  'lists.person_states',
  personStateTemplate,
  personStates
);
```

После выполнения в справочнике «Персонал — Состояния сотрудников» должны
появиться три элемента. Повторный запуск не создаёт дубликаты: элементы с теми
же `id` назначаются заново.

## Другие справочники

Функцию можно использовать без изменения. Замените путь списка и XML-шаблон:

| Справочник | Путь списка | Корневой элемент | `SPXML-FORM-ELEM` |
| --- | --- | --- | --- |
| Персонал — Категории | `categorys` | `category` | `categorys.category` |
| Учебный центр — Организационные формы | `lists.organizational_forms` | `organizational_form` | `lists.organizational_forms.organizational_form` |
| Учебный центр — Формы проведения | `lists.event_forms` | `event_form` | `lists.event_forms.event_form` |
| Оценка персонала — Типы компетенций | `lists.competence_types` | `competence_type` | `lists.competence_types.competence_type` |

Для `categorys` используйте форму `x-local://wtv/wtv_categorys.xmd`, для
остальных строк таблицы — `x-local://wtv/wtv_lists.xmd`.

Например, шаблон категории выглядит так:

```js
var categoryTemplate =
  '<category ' +
  'SPXML-FORM="x-local://wtv/wtv_categorys.xmd" ' +
  'SPXML-FORM-ELEM="categorys.category">' +
  '<id></id><name></name>' +
  '</category>';
```

## Ограничения

- `ms_tools.obtain_shared_list_elem()` — внутренний метод поставки, отдельная
  публичная карточка API для него не найдена. Перед применением на другой
  версии проверьте наличие и сигнатуру метода.
- Метод не возвращает надёжный результат операции и внутри перехватывает часть
  исключений с выводом через `alert()`. После выполнения проверьте каждый
  изменённый справочник в интерфейсе.
- Элемент с существующим ключом назначается заново. Если у типа есть
  дополнительные поля, добавьте их в XML-шаблон и заполните до вызова, иначе
  можно потерять прежние значения.
- Путь списка, имя корневого элемента и `SPXML-FORM-ELEM` зависят от конкретного
  справочника. Не подставляйте значения из недоверенного ввода.

## Источники и проверка

- Реализация `ms_tools.obtain_shared_list_elem()` сопоставлена в файле
  `wtv/ms_tools.xml` поставок `2022.1.3.434`, `2023.2.906` и `2025.1.1333`.
- Штатные вызовы метода найдены в формах `wtv_view_list.xms` тех же поставок.
- Локально проверены структура Markdown и строгая сборка сайта. Изменение
  справочников на стенде WebSoft HCM не выполнялось.
