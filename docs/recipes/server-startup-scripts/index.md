# Как выполнить код при запуске WebSoft HCM Server

Рецепт загружает серверную библиотеку и выполняет обычный JavaScript-файл
при запуске WebSoft HCM Server.

## Условия применения

- Код выполняется на сервере.
- Рецепт проверен в WebSoft HCM Server `2022.1.3.434` и `2023.2.906`.
- Используемая СУБД не имеет значения.
- Для применения изменений требуется перезапуск сервера.

## Как работает решение

При запуске сервер читает `source/api_ext.xml` и загружает
`server_startup.xml`. Его обработчик `OnInit` последовательно:

1. регистрирует v2-библиотеку `global_variables.bs`;
2. выполняет файл `after_start.js`;
3. записывает результат каждого действия в журнал `xhttp`.

Ошибки обрабатываются отдельно, поэтому сбой одного действия не мешает
выполнить следующее.

## Подготовка файлов

Создайте в каталоге сервера следующую структуру:

```text
WebsoftServer/
├─ source/
│  └─ api_ext.xml
└─ wtv/
   └─ _custom_wtv/
      └─ _startup/
         ├─ server_startup.xml
         ├─ global_variables.bs
         └─ after_start.js
```

### Загрузочный файл

Поместите в `server_startup.xml` следующий код:

```xml
<?xml version="1.0" encoding="utf-8"?>
<SPXML-INLINE-FORM>
  <OnInit PROPERTY="1" EXPR="
    try
    {
      RegisterCodeLibrary(
        'x-local://wtv/_custom_wtv/_startup/global_variables.bs'
      );
      alert( '[server_startup] global_variables registered' );
    }
    catch ( err )
    {
      alert( '[server_startup] global_variables failed: ' + err );
    }

    try
    {
      EvalCodeUrl( 'x-local://wtv/_custom_wtv/_startup/after_start.js' );
      alert( '[server_startup] after_start executed' );
    }
    catch ( err )
    {
      alert( '[server_startup] after_start failed: ' + err );
    }
  "/>
</SPXML-INLINE-FORM>
```

### Серверная библиотека

Поместите в `global_variables.bs` следующий код:

```js
"META:NAMESPACE:global_variables";

var _server_mode = 'dev';

function get_server_mode()
{
  return _server_mode;
}
```

После регистрации метод доступен серверному коду как
`global_variables.get_server_mode()`.

### Обычный стартовый скрипт

Поместите в `after_start.js` код, который требуется выполнить один раз при
загрузке `server_startup.xml`. Для проверки достаточно сообщения в журнале:

```js
alert( '[after_start] script body executed' );
```

Стартовый код должен быть идемпотентным: повторное выполнение не должно
создавать дубликаты данных или повреждать состояние.

## Подключение загрузочного файла

Откройте `WebsoftServer/source/api_ext.xml` и добавьте следующий блок `api`
в существующий элемент `apis`:

```xml
<api>
  <name>Server startup scripts</name>
  <libs>
    <lib>
      <path>x-local://wtv/_custom_wtv/_startup/server_startup.xml</path>
    </lib>
  </libs>
</api>
```

Не заменяйте этим фрагментом весь `api_ext.xml`: сохраните остальные блоки
`api`. Если файла ещё нет, минимальный вариант выглядит так:

```xml
<?xml version="1.0" encoding="utf-8"?>
<api_ext>
  <apis>
    <api>
      <name>Server startup scripts</name>
      <libs>
        <lib>
          <path>x-local://wtv/_custom_wtv/_startup/server_startup.xml</path>
        </lib>
      </libs>
    </api>
  </apis>
</api_ext>
```

Перезапустите сервер и проверьте журнал `xhttp`. При успешной загрузке в нём
должны появиться строки:

```text
[server_startup] global_variables registered
[after_start] script body executed
[server_startup] after_start executed
```

Также выполните в серверном коде:

```js
alert( global_variables.get_server_mode() );
```

Ожидаемый результат:

```text
dev
```

Сообщение с суффиксом `failed` означает, что соответствующее действие не
выполнено. Проверьте URL, имя файла и текст ошибки в той же строке журнала.

## Как добавить действия

Добавляйте действия в `server_startup.xml` в требуемом порядке:

- v2-библиотеки `.bs` и `.xmi` регистрируйте через
  `RegisterCodeLibrary()`;
- обычные JavaScript-файлы, код которых нужно выполнить, запускайте через
  `EvalCodeUrl()`;
- независимые действия помещайте в отдельные блоки `try` / `catch`, чтобы
  ошибка одного действия не останавливала остальные.

## Ограничения и безопасность

- Стартовый код выполняется с правами серверного процесса. Ограничьте доступ
  на запись к каталогу `_startup`.
- Перехваченная ошибка не останавливает запуск сервера. Наличие строк
  `failed` нужно контролировать в журнале.
- `OnInit` выполняется при загрузке `server_startup.xml`, а не только при
  запуске операционной системы. Не полагайтесь на однократность без
  идемпотентности действий.
- В кластере загрузка происходит отдельно на каждой ноде. Общие изменения
  должны учитывать параллельное выполнение.

## Файлы

- [server_startup.xml](files/server_startup.xml)
- [global_variables.bs](files/global_variables.bs)
- [after_start.js](files/after_start.js)

## Источники и проверка

- [`RegisterCodeLibrary()`](http://docs.datex.ru/article.htm?id=7172076235998782866)
  — регистрация v2-библиотеки из файла `.bs` или `.xmi`.
- [`EvalCodeUrl()`](http://docs.datex.ru/article.htm?id=5620250451197911782)
  — загрузка и выполнение JavaScript-файла.
- Схема подключения через `api_ext.xml` сопоставлена со штатным примером из
  поставки WebSoft HCM Server `2023.2.906`.
- Пользователь проверил весь рецепт на стендах WebSoft HCM Server
  `2022.1.3.434` и `2023.2.906`.
