# Как определить перезапуск WebSoft HCM Server

Рецепт создаёт идентификатор текущего экземпляра серверной библиотеки и
запоминает время её загрузки. Если сохранённый ранее идентификатор отличается
от текущего, был создан новый экземпляр библиотеки. При подключении только
через `api_ext.xml` это позволяет обнаружить новый запуск сервера.

## Условия применения

- Проверено в WebSoft HCM Server `2022.1.3.434` и `2023.2.906`.
- Код выполняется на сервере.
- Используемая СУБД не имеет значения: состояние хранится в памяти процесса.
- Работа в кластере не проверялась.

## Как работает решение

При запуске сервера `source/api_ext.xml` загружает файл
`server_instance.xml`. Его обработчик `OnInit` регистрирует v2-библиотеку
`server_instance.bs` в глобальном пространстве имён `server_instance`.

Во время регистрации библиотека один раз вычисляет два значения:

- `loaded_at` — время загрузки библиотеки;
- `instance_id` — строковый идентификатор, созданный функцией
  `UniqueStrID()`.

Оба значения остаются неизменными, пока загружен текущий экземпляр
библиотеки.

## Подготовка файлов

Создайте в каталоге сервера следующую структуру:

```text
WebsoftServer/
└─ wtv/
   └─ _custom_wtv/
      └─ _libs/
         └─ server_instance/
            ├─ server_instance.xml
            └─ server_instance.bs
```

Этому каталогу соответствует URL:

```text
x-local://wtv/_custom_wtv/_libs/server_instance/
```

### Загрузочный файл

Поместите в `server_instance.xml` следующий код:

```xml
<?xml version="1.0" encoding="utf-8"?>
<SPXML-INLINE-FORM>
  <OnInit PROPERTY="1" EXPR="
    try
    {
      RegisterCodeLibrary(
        'x-local://wtv/_custom_wtv/_libs/server_instance/server_instance.bs'
      );
      alert( '[server_instance] initialized' );
    }
    catch ( err )
    {
      alert( '[server_instance] initialization failed: ' + err );
    }
  "/>
</SPXML-INLINE-FORM>
```

Обработчик перехватывает ошибку регистрации: она не выходит из `OnInit`, а её
описание записывается в журнал `xhttp`.

### Библиотека

Поместите в `server_instance.bs` следующий код:

```js
"META:NAMESPACE:server_instance";

var _loaded_at = Date();
var _instance_id = UniqueStrID();

function get_info()
{
  return {
    loaded_at: _loaded_at,
    instance_id: _instance_id
  };
}
```

Внешний код получает состояние через `get_info()` и не изменяет внутренние
переменные напрямую.

## Подключение библиотеки

Откройте `WebsoftServer/source/api_ext.xml` и добавьте следующий блок `api`
в существующий элемент `apis`:

```xml
<api>
  <name>Server instance</name>
  <libs>
    <lib>
      <path>x-local://wtv/_custom_wtv/_libs/server_instance/server_instance.xml</path>
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
      <name>Server instance</name>
      <libs>
        <lib>
          <path>x-local://wtv/_custom_wtv/_libs/server_instance/server_instance.xml</path>
        </lib>
      </libs>
    </api>
  </apis>
</api_ext>
```

Перезапустите сервер. В журнале `xhttp` должна появиться строка:

```text
[server_instance] initialized
```

Сообщение `initialization failed` означает, что библиотека не
зарегистрирована. Проверьте URL, имена файлов и текст ошибки в журнале.

## Получение информации

Выполните в серверном коде:

```js
var info = server_instance.get_info();

alert(info.loaded_at);
alert(info.instance_id);
```

Повторные вызовы без перезапуска возвращают те же значения. Запомните
`instance_id`, перезапустите сервер и выполните код ещё раз: новый экземпляр
получит другой идентификатор и более позднее время загрузки.

Для автоматической проверки сравните текущий идентификатор с сохранённым
ранее:

```js
var previous_instance_id = 'идентификатор из предыдущего вызова';
var current_instance_id = server_instance.get_info().instance_id;

if (current_instance_id != previous_instance_id)
{
  alert('Экземпляр серверной библиотеки изменился');
}
```

Место хранения предыдущего значения зависит от потребителя: интеграция может
хранить его у себя, а серверная задача — в собственном постоянном объекте.

## Ограничения

- `loaded_at` — время загрузки библиотеки, а не точное системное время запуска
  процесса. Эти моменты близки только при успешной загрузке через
  `api_ext.xml` во время старта сервера.
- Идентификатор характеризует экземпляр библиотеки. Не регистрируйте её
  повторно в работающем сервере: такое поведение в рамках рецепта не
  проверялось.
- В кластере каждая нода хранит собственное состояние. Без привязки запроса к
  ноде разные идентификаторы нельзя однозначно трактовать как перезапуск.
- Решение не ведёт историю запусков. Для аудита значения нужно сохранять во
  внешнем или постоянном хранилище.

## Файлы

- [server_instance.xml](files/server_instance.xml)
- [server_instance.bs](files/server_instance.bs)

## Источники и проверка

- [`RegisterCodeLibrary()`](http://docs.datex.ru/article.htm?id=7172076235998782866)
  — регистрация v2-библиотеки.
- [`UniqueStrID()`](http://docs.datex.ru/article.htm?id=7172076235998782734)
  — создание строкового идентификатора.
- Схема подключения через `api_ext.xml` сопоставлена со штатным примером из
  поставки WebSoft HCM Server `2023.2.906`.
- Пользователь проверил загрузку библиотеки и получение значений на стендах
  `2022.1.3.434` и `2023.2.906`.
