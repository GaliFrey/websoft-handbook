# HTTP-запросы из серверного кода WebSoft

Готовый код для типовых интеграций и его запуск на локальных стендах —
в [сборнике практических рецептов](../../recipes/http-request/index.md). Ниже — устройство API,
различия сборок и причины ограничений.

Для JSON API используйте `Websoft.HttpRequest.dll` с собственным объектом
запроса и явно созданным `HttpClientHandler`. Так можно отдельно обработать
HTTP-статус и транспортную ошибку, настроить TLS, gzip, proxy и повторное
использование соединений. В сборках 906 и новее можно задать таймаут.

Встроенная функция `HttpRequest()` остаётся полезной для бинарных тел и старого
кода. Однако у неё другой контракт ошибок, особый формат параметров и отличия
между сборками. Замена одного вызова другим без проверки аргументов меняет
поведение интеграции.

Материал относится к **серверному SP-XML Script на Linux**. Проверены сборки
434, 906, 1132, 1333 и 1525. Дополнительно на каждой сборке выполнен весь
применимый набор из 14 практических рецептов. Windows, толстый клиент,
браузерный `fetch` и `XMLHttpRequest` этими результатами не покрываются.

## Быстрый пример: отправить JSON

Пример для **906, 1132, 1333 и 1525**. `url`, `token` и `payload` — адрес API,
токен и объект с данными вашего приложения.

```js
var assembly = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll');
var http = assembly.CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var status = 0;
var text = '';
var headers = 'Content-Type: application/json\nAuthorization: Bearer ' + token + '\n';

try {
    response = http.Open(
        url, 'post', EncodeJson(payload), headers,
        handler, null, 60
    );
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;

    status = response.RespCode;
    text = response.Body;
    if (status < 200 || status >= 300)
        throw 'HTTP ' + status + ': ' + text;

    // У 204 и некоторых других успешных ответов тела нет.
    if (text != '') {
        var data = ParseJson(text);
        // Обработать data.
    }
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

Для **434** замените вызов на пятиаргументный:

```js
response = http.Open(url, 'post', EncodeJson(payload), headers, handler);
```

В SP-XML Script здесь нужен и блок `catch`: форма `try/finally` без `catch`
в проверенном серверном шаблоне 1525 не разбирается.

В `Websoft.HttpRequest.dll` версии `1.22.6.8` из сборки 434 нет аргументов
`client` и `timeout`. В эксперименте дополнительные аргументы не вызвали ошибки,
но таймаут `1` был проигнорирован: ответ пришёл через две секунды. Успешный вызов
с семью аргументами не подтверждает их поддержку.

В примере намеренно указан `application/json` без `; charset=utf-8`:
DLL всё равно кодирует строку тела в UTF-8, а сборки 434–1132 не принимают
параметры в этом заголовке. Настройка gzip описана [ниже](#gzip).

## Два API с похожими названиями

| Свойство | Встроенный `HttpRequest()` | `Websoft.HttpRequest.HttpRequest.Open()` |
| --- | --- | --- |
| Получение | Глобальная функция SP-XML | .NET-объект из DLL |
| Тело запроса | Строка, в том числе бинарные данные | Строка, которая преобразуется в UTF-8 |
| HTTP 201, 204, 400 | В тестах — исключение без `Ignore-Errors` | Объект ответа с фактическим `RespCode` |
| Ошибка соединения | Исключение | Обычно пустой результат и `Error` / `ErrorMessage`; возможны также исключения моста SP-XML/.NET |
| Заголовки ответа | `Header`, объект | `Headers`, строка JSON |
| Адрес ответа | `Url` заполнен | В исследованных реализациях `Url` не присваивается |
| Бинарный ответ | Надёжно проверено сохранение через `SaveToFile()` | Проверен `BinaryBody` |

`Headers` DLL содержит заголовки `HttpResponseMessage.Headers`, но не
`Content.Headers`. Например, `Content-Type` в нём отсутствует. Несколько
`Set-Cookie` объединяются в строку; для сессионной авторизации используйте
cookie-контейнер handler, а не разбор этой строки по пробелам.

### Зачем создавать свой объект

Обычный способ подключения тоже работает:

```js
var http = tools.get_object_assembly('HttpRequest');
```

Но загрузчик в 434, 906, 1333 и 1525 кэширует экземпляр объекта. В 1132 для
него задано `bNoCache: true`, поэтому создаются новые экземпляры. Поля
`Error` и `ErrorMessage` изменяются каждым вызовом `Open`. При параллельном
использовании общего объекта это создаёт риск чтения чужого результата ошибки.
Риск следует из кода; отдельный конкурентный стресс-тест не выполнялся.

`GetAssembly(...).CreateClassObject(...)` из основного примера создаёт собственную
обёртку независимо от этой настройки. Менять коробочный `bNoCache` для
интеграции не требуется. Кэширование обёртки **не означает** кэширование
внутреннего `HttpClient` или его соединений.

## Аргументы Open и различия сборок

```js
http.Open(url, method, body, headers, handler, client, timeout, jsonMultipartContent)
```

| Аргумент | Назначение |
| --- | --- |
| `url` | Полный адрес запроса |
| `method` | Метод, например `get`, `post`, `put`, `patch`, `delete`; регистр в DLL несущественен |
| `body` | Строковое тело; для JSON используйте `EncodeJson`, для формы — `UrlEncodeQuery` |
| `headers` | Строка `Имя: значение\n` |
| `handler` | `HttpClientHandler`: TLS, proxy, cookies, редиректы, gzip, соединения |
| `client` | Готовый `HttpClient`; для обычного вызова передавайте `null` |
| `timeout` | Положительное число секунд; `0` не изменяет таймаут клиента |
| `jsonMultipartContent` | JSON-описание частей multipart, доступно с 1333 |

| Сборка | Аргументов Open | Таймаут | Значение заголовка с `:` | `Content-Type` с параметрами | Multipart через восьмой аргумент |
| --- | ---: | --- | --- | --- | --- |
| 434 | 5 | Нет аргумента | Обрезается | Ошибка | Нет |
| 906 | 7 | Да | Сохраняется | Ошибка | Нет |
| 1132 | 7 | Да | Сохраняется | Ошибка | Нет |
| 1333 | 8 | Да | Сохраняется | Да | Да |
| 1525 | 8 | Да | Сохраняется | Да | Да |

Если `client == null`, DLL создаёт клиент на время одного `Open` и освобождает
его перед возвратом. Переданный handler при этом не освобождается — он остаётся
владельцу. Если передан `client`, DLL использует именно его; отдельный аргумент
`handler` не перенастраивает уже существующий клиент.

## Заголовки, JSON и авторизация

```js
var jsonHeaders = 'Content-Type: application/json\n';
var bearer = 'Authorization: Bearer ' + token + '\n';
var basic = 'Authorization: Basic ' + Base64Encode(login + ':' + password) + '\n';
var form = UrlEncodeQuery({name: 'Иван', value: 'a+b'});
```

Сериализуйте JSON один раз. Не заменяйте `EncodeJson` на `UrlEncodeQuery`,
чтобы устранить ошибку `201 Created`: это меняет формат тела. Сначала
проверьте HTTP-статус и способ его обработки.

У DLL пустая строка `body` означает отсутствие `HttpContent`. Поэтому переданный
`Content-Type` для пустого тела не появился в фактическом запросе. Если API
ожидает JSON-объект, отправьте `'{}'`; не подменяйте этим действительно пустое
тело, если контракт API требует именно его.

### Если используется встроенная функция

```js
var response = HttpRequest(
    url, 'post', EncodeJson(payload),
    'Content-Type: application/json\nIgnore-Errors: 1\n'
);
var status = response.RespCode;
if (status < 200 || status >= 300)
    throw 'HTTP ' + status + ': ' + response.Body;
```

**Завершайте каждую строку, включая последнюю, символом `\n`.** На всех
проверенных сборках последний параметр без перевода строки не применился.
Это касается и `Content-Type`, и `Ignore-Errors`, и `Auto-Redirect`.

`Ignore-Errors` позволяет получить HTTP-ответ для проверки статуса. Он не
превращает ошибку соединения в успешный ответ. Для запрета редиректов и чтения
статуса 302 нужны оба параметра:

```js
'Auto-Redirect: 0\nIgnore-Errors: 1\n'
```

Эти параметры относятся к встроенному вызову. В DLL они не управляют
поведением клиента и могут отправляться как обычные HTTP-заголовки.
Для DLL задавайте `handler.AllowAutoRedirect = false` **до первого запроса**.

Встроенная функция также использует состояние, заданное `SetHttpDefaultAuth`.
Коробочные сценарии обмена данными вызывают эту функцию. Если на принимающей
стороне появился лишний `Authorization`, проверяйте этот путь и фактические
заголовки. Не добавляйте безусловный сброс глобальной авторизации в каждый
агент: он может затронуть другой код.

## Таймауты и серии запросов

У DLL 906+ с `client = null` можно задавать таймаут каждому вызову:

```js
response = http.Open(url, 'get', '', '', handler, null, 60);
```

Проверка с ответом через две секунды и таймаутом `1` завершилась транспортной
ошибкой примерно через секунду. Это ограничение операции клиента, а не только
установления TCP-соединения.

В наших конфигурациях `TCP-CONN-TIMEOUT` и `TCP-SRV-TIMEOUT` равны `3600000`.
Обе реализации дождались ответа с задержкой 35 секунд. На 434 также проверена
пауза 35 секунд после начала тела ответа. Эти опыты не устанавливают максимальный
таймаут и не подтверждают универсального ограничения встроенной функции в
30 секунд. Влияние изменения INI отдельно не проверялось.

### Повторно используйте handler

Для последовательной пачки запросов создайте один `http` и один `handler`
до цикла. В каждом вызове передавайте этот handler и `client = null`,
освобождайте каждый ответ, а handler — после завершения всей пачки.

В тесте из пяти запросов:

- DLL без общего handler открывала пять соединений;
- DLL с общим handler использовала одно соединение;
- встроенный вызов также использовал одно соединение.

Handler сохраняет не только соединения, но и cookies. Используйте отдельный
handler для каждой независимой сессии или интеграции. Для REST API, которому
cookies не нужны, их можно отключить через `UseCookies` до первого запроса;
этот режим проверен в практических рецептах на всех пяти сборках.

### Сессионная авторизация и cookies

`CreateNewHttpClientHandler(false)` возвращает стандартный .NET
`HttpClientHandler`. При `UseCookies = true` его `CookieContainer` автоматически
принимает `Set-Cookie` из ответа и добавляет подходящий заголовок `Cookie` в
следующие запросы к тому же домену и пути. Это значение включено по умолчанию,
но в коде с сессионной авторизацией его полезно задавать явно.

Для сборки 434 запрос входа и следующий запрос выполняются так. `loginUrl`,
`profileUrl`, `login` и `password` задаёт интеграция:

```js
var assembly = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll');
var http = assembly.CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
handler.UseCookies = true;

var response = null;
var loginBody = UrlEncodeQuery({login: login, password: password});
var loginHeaders = 'Content-Type: application/x-www-form-urlencoded\n';
var result = null;

try {
    response = http.Open(loginUrl, 'post', loginBody, loginHeaders, handler);
    if (response == undefined || response == null)
        throw 'Login transport error: ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'Login HTTP ' + response.RespCode + ': ' + response.Body;

    response.Dispose();
    response = null;

    // Тот же handler автоматически отправит сохранённые cookies.
    response = http.Open(profileUrl, 'get', '', '', handler);
    if (response == undefined || response == null)
        throw 'Request transport error: ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'Request HTTP ' + response.RespCode + ': ' + response.Body;

    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

В сборках 906 и новее оба вызова могут использовать `client = null` и таймаут:

```js
response = http.Open(loginUrl, 'post', loginBody, loginHeaders, handler, null, 60);
response = http.Open(profileUrl, 'get', '', '', handler, null, 60);
```

Не извлекайте cookies из `response.Headers` и не разбирайте объединённый
`Set-Cookie` по пробелам. Контейнер учитывает домен, путь, `Secure` и срок
действия. Новый handler создаёт новую пустую сессию; общий handler смешивает
cookies всех использующих его запросов, поэтому для независимых пользователей
или интеграций нужны отдельные экземпляры.

Автоматическая работа контейнера проверена на всех пяти сборках: первый запрос
приходил без `Cookie`, а запросы со второго по пятый содержали cookies, заданные
первым ответом. Контракт свойств описан в документации Microsoft:
[CookieContainer](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclienthandler.cookiecontainer?view=net-10.0)
и [UseCookies](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclienthandler.usecookies?view=net-10.0).

Это проверка повторного использования соединений, **не нагрузочный тест на
100 тысяч запросов**. Для большого потока дополнительно нужны ограниченная
конкурентность, обработка 429/503 и политика повторов по контракту удалённого API.
Не повторяйте автоматически POST после таймаута: сервер мог уже выполнить операцию.

### Почему готовый HttpClient требует осторожности

Начиная с 906 можно создать `http.CreateNewHttpClient(handler)` и передавать его
шестым аргументом. Но в этой обёртке есть два существенных ограничения:

1. Положительный `timeout` присваивается свойству `client.Timeout` при каждом
   `Open`. После первого запроса .NET запрещает такое изменение; второй вызов
   возвращает пустой результат с сообщением `Properties can only be modified
   before sending the first request`.
2. Заголовки добавляются в `DefaultRequestHeaders`. В последовательных запросах
   `X-Sequence: 0`, затем `1`, затем `2` сервер получил `0`, `0, 1`, `0, 1, 2`.

В последовательном коде помогают настройка таймаута до первого запроса,
`timeout = 0` в следующих вызовах и `client.DefaultRequestHeaders.Clear()`
перед новым набором заголовков. Очистка общего клиента при параллельных запросах
небезопасна. Для большинства интеграций вариант с общим handler проще.

Повторное использование соединений соответствует
[рекомендациям Microsoft по HttpClient](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/httpclient-guidelines).

## Gzip

Встроенная функция автоматически запрашивает сжатие. Контрольный gzip-ответ
прочитан в 434 и 906; в 1132, 1333 и 1525 возникла ошибка
`Unable to decode Gzip content encoding`.

DLL без настройки handler не распаковывает принудительно сжатый ответ.
Если такой ответ прочитать через `Body`, это не будет JSON. Настройте распаковку
перед первым запросом:

```js
var systemAssembly = tools.dotnet_host.Object.GetAssembly('System.Private.CoreLib.dll');
var enumType = systemAssembly.CallClassStaticMethod(
    'System.Type', 'GetType',
    ['System.Net.DecompressionMethods, System.Net.Primitives']
);
handler.AutomaticDecompression = systemAssembly.CallClassStaticMethod(
    'System.Enum', 'ToObject', [enumType, 3]
);
```

`3` означает сочетание GZip и Deflate. Здесь нужен типизированный .NET enum:
прямое присваивание числа `handler.AutomaticDecompression = 3` на 434
завершилось ошибкой моста. Приведённый вариант распаковал контрольный gzip.
Deflate и Brotli отдельными ответами не проверялись.

## HTTPS и proxy

Создавайте handler через `CreateNewHttpClientHandler(false)`. Несмотря на имя
аргумента `ignoreClientCertficate`, значение `true` в исследованном коде
отключает проверку **сертификата сервера**. Оно не подключает клиентский
сертификат для mTLS.

В тесте с новым самоподписанным сертификатом DLL по умолчанию отклонила
соединение, а встроенный вызов на наших Linux-стендах его принял. Поэтому
успешный встроенный HTTPS-запрос в этих условиях не доказывает подлинность
сервера. Для HTTPS-интеграций выбирайте DLL с включённой проверкой и корректной
цепочкой доверия. Windows, корпоративные CA и mTLS требуют отдельной проверки.

Явный HTTP-proxy можно настроить на handler без изменения общих INI:

```js
var uriAssembly = tools.dotnet_host.Object.GetAssembly('System.Private.Uri.dll');
var proxyUri = uriAssembly.CreateClassObject('System.UriBuilder');
proxyUri.Scheme = 'http';
proxyUri.Host = 'proxy.example.org';
proxyUri.Port = 3128;

var proxyAssembly = tools.dotnet_host.Object.GetAssembly('System.Net.WebProxy.dll');
var proxy = proxyAssembly.CreateClassObject('System.Net.WebProxy');
proxy.Address = proxyUri.Uri;
handler.Proxy = proxy;
handler.UseProxy = true;
```

Проверена доставка HTTP-запроса тестовому proxy с абсолютным адресом назначения.
HTTPS CONNECT, proxy с паролем, NTLM/Kerberos, системные настройки proxy и
поведение `HTTP-USE-WININET` на Windows в эту проверку не входили.

## Файлы и multipart

### Скачать файл

Для встроенного ответа используйте:

```js
response.SaveToFile('/tmp/download.bin');
```

Контрольные байты `00 01 7F 80 FF 0D 0A` сохранились без изменений.
`Base64Encode(response.BinaryBody)` у встроенного ответа в том же тесте вернул
пустую строку. Это наблюдение о данном пути чтения, а не доказательство потери
самого HTTP-тела.

У DLL проверено получение тех же байтов через `BinaryBody`. Для сохранения:

```js
PutFileData('/tmp/download.bin', response.BinaryBody);
```

Ответ необходимо освободить через `Dispose()` после чтения или сохранения.
`Body` предназначен для текста, а не произвольных байтов.

### Отправить бинарное тело

Встроенному `HttpRequest` можно передать `LoadFileData(path)` с нужным
`Content-Type`. Это сохранило контрольные байты. Учитывайте ограничение проверки
TLS, описанное выше.

В `Open` DLL аргумент `body` имеет тип `string`; затем выполняется преобразование
в UTF-8. В тесте семь исходных байтов превратились в одиннадцать, а `80` и `FF`
были заменены. **Не передавайте произвольный файл в строковый body DLL.**
ASCII-файл может случайно пройти такую проверку, поэтому проверяйте бинарный
образец с нулевым байтом и значениями выше `7F`.

### Multipart в 1333 и 1525

Для этих DLL есть отдельный восьмой аргумент. Файлы указываются путями
**в файловой системе процесса WebSoft**, а не `x-local://` URL:

```js
var parts = [
    {Type: 'field', FieldName: 'note', Value: 'Привет'},
    {
        Type: 'file', FieldName: 'file',
        FilePath: '/tmp/sample.bin', FileName: 'sample.bin',
        ContentType: 'application/octet-stream'
    }
];
response = http.Open(
    url, 'post', '', 'Authorization: Bearer ' + token + '\n',
    handler, null, 60, EncodeJson(parts)
);
```

Boundary и `Content-Type: multipart/form-data` формирует DLL. В этой ветви код
вызывает `PostAsync` независимо от `method`: применять её для multipart PUT
нельзя. В исходниках также нет явного освобождения созданного multipart и
открытых файловых потоков. `response.Dispose()` освобождает содержимое ответа,
но не эти части запроса. Перед массовой отправкой файлов нужен отдельный
контроль файловых дескрипторов; такая нагрузка здесь не проверялась.

### Multipart через встроенную функцию

`MultipartFormEncode` формирует строку, начинающуюся с собственного
`Content-Type` и пустой строки. Отделите их от тела. Пример для фиксированных
полей `note` и `file` и фиксированного безопасного имени файла:

```js
var body = MultipartFormEncode(
    'note', 'Привет', 'file', LoadFileData('/tmp/sample.bin')
);
var separator = body.indexOf('\r\n\r\n');
var contentHeader = StrLeftRange(body, separator);
body = StrRightRangePos(body, separator + 4);
body = StrReplace(
    body, 'name="file"',
    'name="file"; filename="sample.bin"\r\nContent-Type: application/octet-stream'
);
response = HttpRequest(url, 'post', body, contentHeader + '\nIgnore-Errors: 1\n');
```

Этот пример проверен с одним бинарным файлом. Это не универсальный сериализатор:
нельзя без экранирования подставлять произвольные имена файлов и полей,
а глобальная замена может задеть совпавшую последовательность в данных.
Для нескольких файлов, сложных метаданных и массовой передачи предпочтителен
отдельный .NET-адаптер с корректным освобождением потоков.

## Что проверять при ошибке

| Симптом | Первое действие |
| --- | --- |
| `undefined` после `.Open()` | Сразу прочитать `Error` и `ErrorMessage` у своего объекта; отдельно перехватить исключение моста |
| `HTTP 201 Created` выглядит как ошибка | Для встроенной функции добавить `Ignore-Errors: 1\n`, затем проверить статус |
| В Postman работает, в WebSoft нет | Сопоставить фактические URL, метод, заголовки и байты; учесть разные окружения и trust store |
| Заголовок отсутствует | Проверить завершающий `\n`; в DLL проверить, не относится ли заголовок к содержимому и не пусто ли тело |
| Заголовок обрезан по `:` | Проверить DLL 434; встроенный вызов и DLL 906+ прошли контрольный пример |
| `Content-Type` с charset вызывает ошибку | Для DLL 434–1132 передавать `application/json` без параметров |
| Gzip нельзя прочитать | Включить распаковку handler; проверить сборку встроенного вызова |
| Второй запрос с общим клиентом падает | Проверить повторное присваивание таймаута |
| Заголовки дублируются | Проверить общий `DefaultRequestHeaders` или `SetHttpDefaultAuth` встроенного клиента |
| Файл меняется или обрезается | Проверить точные байты; не использовать строковый body DLL для бинарного файла |

Логируйте метод, адрес без секретных параметров, длительность, HTTP-статус
и текст транспортной ошибки. Не записывайте токены, пароли и содержимое
персональных документов в диагностический журнал.

## Итоговая совместимость

| Проверенный сценарий | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| JSON через DLL без параметров Content-Type | Да¹ | Да | Да | Да | Да |
| Gzip встроенным вызовом | Да | Да | Ошибка | Ошибка | Ошибка |
| Gzip через настроенный handler DLL | Да | Да | Да | Да | Да |
| Бинарная загрузка через встроенный вызов | Да | Да | Да | Да | Да |
| Бинарная загрузка через строковый body DLL | Повреждение | Повреждение | Повреждение | Повреждение | Повреждение |
| Встроенный multipart из примера | Да | Да | Да | Да | Да |
| Multipart POST через восьмой аргумент DLL | Нет API | Нет API | Нет API | Да | Да |
| 12 МиБ в обе стороны, обе реализации | Да | Да | Да | Да | Да |
| Ответ после паузы 35 секунд, обе реализации | Да | Да | Да | Да | Да |
| Самоподписанный TLS-сертификат, встроенный вызов | Принят | Принят | Принят | Принят | Принят |
| Самоподписанный TLS-сертификат, DLL по умолчанию | Отклонён | Отклонён | Отклонён | Отклонён | Отклонён |

¹ В 434 JSON-тело передано правильно, но контрольный заголовок даты с
двоеточиями обрезался. Эта оговорка не относится к самому JSON.

Идентификация DLL важна даже при одинаковом номере HCM:

| HCM Server | Версия Websoft.HttpRequest.dll |
| --- | --- |
| 2022.1.3.434 | 1.22.6.8 |
| 2023.2.906 | 1.24.5.14 |
| 2025.1.1132 | 1.25.2.21 |
| 2025.1.1333 | 1.25.7.3 |
| 2025.2.1525 | 2.26.9.7 |

## Условия и границы проверки

Эксперименты выполнены 21 сентября 2026 года на локальных Docker-стендах.
Запросы выполнял серверный SP-XML-обработчик; отдельный Node.js-сервис фиксировал
метод, сырые заголовки, SHA-256 и Base64 тела, а также идентификатор соединения.
Сравнивались ответы SP-XML и записи принимающей стороны. Установленные DLL
сверялись с локальными исходными материалами по SHA-256.

Проверены GET/POST/PUT/PATCH/DELETE, JSON и form, статусы 201/204/400,
редиректы 302/307, gzip, бинарные данные, тела 12 МиБ, задержки, ошибка TLS,
серии запросов, а также описанные варианты multipart и proxy.
У исходящего тела 12 МиБ проверены длина и полный SHA-256, у ответа 12 МиБ —
длина и начало текста. Небольшой бинарный образец проверен побайтно.
Эти опыты не определяют максимальный размер, предел нагрузки или максимальный
таймаут.

Полные локальные запросы, ответы и сведения о сборках находятся в
`references/http-request/` и не публикуются вместе с проприетарными материалами.

Документированные сигнатуры и назначение базовых функций можно сверить с
[HttpRequest в Datex](http://docs.datex.ru/article.htm?id=5620276892448878634)
и [get_object_assembly в WebSoft](https://docs.websoft.ru/_wt/6857020682929111618/parent_id/6809298112856471128).
