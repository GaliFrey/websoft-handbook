# WebSocket 434: закрытие соединений

Действие `close_socket` предназначено для удаления записи из хранилища подписчиков, но в штатной реализации сборки 434 не работает: код обращается к необъявленному `context`. Команда, фактические ошибки и их разбор приведены в [главе о группах](groups.md#unsubscribe). Закрытие соединения и удаление записи о подписчике — разные действия.

Обработчик `close(context)` вызывается при завершении соединения, но не выполняет отписку через `main_socket`. Он пытается вызвать `WriteToWebSocketMessage`, которого нет в Datex.XHTTP `1.22.6.9`; этот вызов нельзя использовать как подтверждение получения сообщения клиентом.

## Abort

В DLL сборки 434 для локального сокета доступен `Abort()` базового `WebSocket`. У удалённого `VirtualWebSocket` этот метод пустой. На тестовом стенде 434 вызов `Abort()` для локального `System.Net.WebSockets.ManagedWebSocket` также не завершил соединение корректно.

Создадим новое подключение

```
ws://localhost:80/services/main_ws_service?X-StatefulSocketId=555
```

Укажите полный ID тестового соединения и запустите агент:

``` javascript
var xHttpStaticAssembly = tools.get_object_assembly( 'XHTTPMiddlewareStatic' );
var arrWebSockets = xHttpStaticAssembly.CallClassStaticMethod(
    'Datex.XHTTP.WebSocketContext',
    'GetWebSockets'
).ToArray();

var socket;
for (socket in arrWebSockets) {
    if (socket.Key != '/services/main_ws_service-s-555') continue;
    CallObjectMethod(socket.Value, 'Abort', []);
}
```

На стенде получен следующий результат:

- до `Abort()` клиент получил контрольное сообщение;
- после `Abort()` ID остался в `GetWebSockets()`;
- браузер продолжил показывать соединение как активное;
- контрольное сообщение после `Abort()` клиенту не доставлено;
- обработчик `close(context)` не вызван.

Поэтому `Abort()` нельзя использовать как надёжный способ закрытия WebSocket в сборке 434.

[К обзору сборки 434](index.md).
