# WebSocket 906: теги

[Перед началом: подготовка и подключение](connection.md).

В WebSoft HCM Server `2023.2.906` теги предоставляет Datex.XHTTP `1.24.4.27`.

Назначение тегов сообщением сервису и отбор группы описаны в [главе о группах](groups.md).

Служебные теги `user_id`, `socket_type`, `pong` используются логикой сервиса; их удаление или замена меняет его поведение.

Команда `init_socket` также заменяет весь набор тегов. Для изменения одной метки без потери остальных используйте `AddOrUpdateTag`. Значения из клиентского `socket_tags` могут перезаписать служебные поля, поэтому проверяйте права по серверному контексту пользователя, а не по тегам.

## Подготовка тестового соединения

Откройте отдельное соединение:

```
ws://localhost:80/services/main_ws_service?X-StatefulSocketId=tags-test
```

Отправьте из клиента команду:

``` json
{
    "socket_action": "init_socket",
    "socket_type": "tags_test",
    "uid": "tags-test-init",
    "socket_tags": [
        {"name": "webinar_id", "value": "100"},
        {"name": "role", "value": "listener"}
    ]
}
```

После `init_socket` соединение `/services/main_ws_service-s-tags-test` получит следующий начальный набор тегов:

| Тег | Значение |
| --- | --- |
| `user_id` | ID текущего пользователя в виде строки |
| `socket_type` | `tags_test` |
| `pong` | `"1"` |
| `webinar_id` | `"100"` |
| `role` | `"listener"` |

Выполняйте следующие разделы по порядку: каждый агент получает общий список соединений, но изменяет только `/services/main_ws_service-s-tags-test`, а некоторые примеры меняют состояние для следующего шага. Код каждого раздела поместите в отдельный агент и запустите на изолированном тестовом стенде.

## GetTag

Возвращает значение тега или `null`, если тег отсутствует.

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var arrWebSockets = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetWebSockets',
    [null, false]
).ToArray();
var targetSocketId = '/services/main_ws_service-s-tags-test';
var socket;
var socket_type;

for (socket in arrWebSockets) {
    if (socket.Key != targetSocketId) continue;
    // Получаем значение тега по имени
    socket_type = socket.Value.GetTag("socket_type");
    alert(socket_type);
}
```

```
16:33:13 [0469] tags_test
```

## RemoveTag

Удаляет один тег по имени и возвращает признак успешного удаления.

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var arrWebSockets = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetWebSockets',
    [null, false]
).ToArray();
var targetSocketId = '/services/main_ws_service-s-tags-test';
var socket;
var tag;

for (socket in arrWebSockets) {
    if (socket.Key != targetSocketId) continue;
    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }

    // Удаляем тег
    socket.Value.RemoveTag("socket_type");

    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }
}
```

```
16:33:49 [0737] pong: 1
16:33:49 [0737] user_id: 6148914691236517121
16:33:49 [0737] role: listener
16:33:49 [0737] webinar_id: 100
16:33:49 [0737] socket_type: tags_test

16:33:49 [0737] pong: 1
16:33:49 [0737] user_id: 6148914691236517121
16:33:49 [0737] role: listener
16:33:49 [0737] webinar_id: 100
```

## SetTagsFromJson

Заменяет набор тегов свойствами непустого JSON-объекта. В `1.24.4.27` передача `{}` не очищает теги: для полного удаления нужно пройти по `Tags` и вызвать `RemoveTag` для каждого имени.

JSON-числа преобразуются в `double`. Идентификаторы, для которых важна точность всех цифр, передавайте строками. Вложенные объекты и массивы остаются `JsonElement`.

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var arrWebSockets = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetWebSockets',
    [null, false]
).ToArray();
var targetSocketId = '/services/main_ws_service-s-tags-test';
var socket;
var tag;
var oNewTags;

for (socket in arrWebSockets) {
    if (socket.Key != targetSocketId) continue;
    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }

    oNewTags = new Object();
    oNewTags.webinar_id = "123";
    oNewTags.role = "speaker";
    oNewTags.socket_type = "ws_group_replace";

    // Преобразуем в JSON и заменяем теги. Старые теги удаляются полностью
    socket.Value.SetTagsFromJson( EncodeJson(oNewTags) );

    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }
}
```

```
16:34:40 [0206] pong: 1
16:34:40 [0206] user_id: 6148914691236517121
16:34:40 [0206] role: listener
16:34:40 [0206] webinar_id: 100
16:34:40 [0206] role: speaker
16:34:40 [0206] webinar_id: 123
16:34:40 [0206] socket_type: ws_group_replace
```

## AddOrUpdateTag

Добавляет или заменяет один тег, не удаляя остальные.

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var arrWebSockets = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetWebSockets',
    [null, false]
).ToArray();
var targetSocketId = '/services/main_ws_service-s-tags-test';
var socket;
var tag;

for (socket in arrWebSockets) {
    if (socket.Key != targetSocketId) continue;
    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }

    // Добавляем новый тег
    socket.Value.AddOrUpdateTag("new_tag", "OK!");

    // Заменяем значение существующего тега
    socket.Value.AddOrUpdateTag("socket_type", "repl_ws_group");

    for (tag in socket.Value.Tags) {
        alert(tag.Key + ": " + tag.Value);
    }
}
```

```
16:35:10 [0632] role: speaker
16:35:10 [0632] webinar_id: 123
16:35:10 [0632] socket_type: ws_group_replace
16:35:10 [0632] role: speaker
16:35:10 [0632] new_tag: OK!
16:35:10 [0632] webinar_id: 123
16:35:10 [0632] socket_type: repl_ws_group
```

[Далее: закрытие соединений](closing.md).
