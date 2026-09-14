# WebSocket 906, 1132, 1333 и 1525: закрытие соединений

Примеры этой главы проверены на стендах 906 и 1525. Методы `Abort`,
`GetWebSockets` и `GetLocalWebSocket`, используемые ниже, доступны во всех
рассматриваемых сборках.

Команда `{"socket_action":"close_socket"}` в `main_socket` не закрывает
соединение и не удаляет его теги: ветка обработчика пуста во всех
сборках 906, 1132, 1333 и 1525. На стенде 1525 после этой команды соединение
осталось активным и ответило `$ok` на heartbeat.

Обработчик `close(context)` одинаков в сборках 906, 1132, 1333 и 1525. При
завершении соединения он ставит `check_socket` в очередь методом, который не
возвращает результат доставки. Этот вызов нельзя использовать как подтверждение
получения сообщения клиентом.

## Закрытие со стороны клиента

Кнопка «Отключиться» тестового клиента вызывает
`ws.close(1000, 'manual')`. На стенде 1525 браузер получил событие:

```text
Закрыто: code=1006, reason=
```

Таким образом, переданные клиентом код `1000` и причина `manual` не
подтвердились в результате обмена кадрами закрытия. Код `1006` означает, что
браузер не получил корректный ответный кадр закрытия.

Статический анализ Datex.XHTTP `2.26.8.20` показывает возможную причину:
после входящего кадра закрытия сервер вызывает `CloseAsync` только при состоянии
сокета `Open`, хотя сокет уже может находиться в состоянии `CloseReceived`.
Сетевые кадры в этом опыте не снимались, поэтому это объяснение остаётся
выводом по коду. На стендах 906, 1132 и 1333 этот сценарий отдельно не
проверялся.

## Abort

Метод прерывает соединение, а не выполняет согласованное закрытие через `CloseAsync`. В Datex.XHTTP `1.24.4.27` локальный `DatexWebSocket.Abort()` вызывает `Abort()` базового сокета. У удалённого `DatexVirtualWebSocket` метод пустой, поэтому таким способом нельзя закрыть удалённое соединение.

Откройте отдельное тестовое соединение:

```
ws://localhost:80/services/main_ws_service?X-StatefulSocketId=555
```

Сначала создайте и запустите агент для проверки доставки:

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var socketId = '/services/main_ws_service-s-555';

xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'WriteToWebSocketMessageQueue',
    [ socketId, 'Сообщение до Abort', false ]
);
```

![Сообщение до Abort](../img/906/906-pref-abort.png)

Убедитесь, что клиент получил сообщение. Затем запустите отдельный агент для `Abort()`:

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var socket = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetLocalWebSocket',
    ['/services/main_ws_service-s-555']
);

if (socket == null || socket == undefined) {
    alert('Сокет не найден');
} else {
    CallObjectMethod(socket, 'Abort', []);
    alert('Abort вызван');
}
```

```
16:52:47 [0464] Abort вызван
```

![Закрытие соединения через Abort](../img/906/906-abort.png)

После этого запустите третий агент. Он проверит наличие ID и попробует отправить ещё одно сообщение:

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var socketId = '/services/main_ws_service-s-555';
var socket = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetLocalWebSocket',
    [socketId]
);

if (socket == null || socket == undefined) {
    alert('Сокет удалён из локального списка');
} else {
    alert('Сокет остался в локальном списке');
}

xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'WriteToWebSocketMessageQueue',
    [ socketId, 'Сообщение после Abort', false ]
);
```

```
16:55:28 [0394] Сокет удалён из локального списка
```

На стенде 906 подтверждено:

- до `Abort()` клиент получил контрольное сообщение;
- после `Abort()` браузер сообщил `Закрыто: code=1006, reason=`;
- ID `/services/main_ws_service-s-555` исчез из `GetWebSockets()`;
- контрольное сообщение после `Abort()` клиенту не доставлено.

Код `1006` означает аварийное завершение без кадра закрытия WebSocket. Поэтому `Abort()` в сборке 906 работает как принудительное прерывание локального соединения, но не заменяет согласованное закрытие с кодом `1000` и причиной.

На стенде 1525 после `Abort()` браузер также сообщил
`Закрыто: code=1006, reason=`, а `GetLocalWebSocket` перестал находить соединение.
Доставка сообщений до и после `Abort()` в этом опыте не проверялась.

Это отличается от проверенного поведения сборки 434: там вызов `Abort()` оставил ID в `GetWebSockets()`, а браузер продолжил считать соединение активным.

[К обзору сборок 906, 1132, 1333 и 1525](index.md).
