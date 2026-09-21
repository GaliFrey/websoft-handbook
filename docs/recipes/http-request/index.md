# HTTP: практические рецепты

Здесь собраны примеры серверного SP-XML Script для интеграций: отправка данных,
авторизация, ошибки, файлы и управление соединениями. Каждый отмеченный рецепт
исполняется из **этого Markdown-файла**: тестовый запуск не использует отдельную
копию примера. Объяснение различий API — в
[основной статье](../../guides/http-request/index.md).

## Как пользоваться

Для ручного запуска из серверного агента используйте
[инструкцию ниже](#manual-agent). Для своей интеграции задайте входные переменные,
скопируйте один рецепт и замените учебный маршрут на маршрут вашего API. Результат
примера записывается в `result`; дальнейшие действия с ним относятся к вашему
приложению.

```js
var apiBase = 'http://http-peer:18991'; // В интеграции — HTTPS-адрес вашего API.
var is434 = false;                    // Флаг совместимости: true для 434, false для 906+.
var token = 'test-token';             // В интеграции — из защищённой настройки.
var username = 'demo';
var password = 'p+a&ss';
var inputFile = '/WebsoftServer/http-recipe-binary.bin';
var outputFile = '/tmp/http-recipe-download.bin';
var proxyHost = 'http-peer';
var proxyPort = 18991;
```

`http-peer`, `/echo`, `/status`, `/binary`, `/gzip`, `/delay` и `/redirect` —
маршруты локального тестового сервиса, а не API WebSoft. Сервис возвращает
полученные данные, поэтому учебный токен виден в ответе. Реальные секреты
не подставляйте в тесты и не выводите в журналы. Basic допустим только поверх
доверенного HTTPS; локальный пример использует фиктивные учётные данные.

### Зачем нужна `is434`

`is434` — не переменная WebSoft и не свойство DLL. Это введённый в рецептах
флаг совместимости, который нужно задать вручную перед запуском кода:

- `true` — для сборки 434;
- `false` — для проверенных сборок 906, 1132, 1333 и 1525.

В 434 метод `Open` принимает пять аргументов. В 906 и более новых проверенных
сборках рецепты передают ещё `client = null` и таймаут `15` секунд. Поэтому один
и тот же рецепт содержит две формы вызова:

```js
if (is434) response = http.Open(url, 'get', '', '', handler);
else response = http.Open(url, 'get', '', '', handler, null, 15);
```

Флаг не определяет версию автоматически и не делает непроверенную DLL
совместимой с рецептом. В постоянном коде под одну известную сборку переменную
и лишнюю ветку можно удалить, оставив подходящий вызов `Open`.

Свой экземпляр обёртки и handler принадлежит одному выполнению рецепта. Все
ответы DLL освобождаются в `finally`.
Переменные тела, URL и заголовков объявлены до циклов: повторное объявление
`var` внутри тела цикла в проверенном шаблоне 434 вызвало ошибку на второй итерации.

## Проверить вручную из агента WebSoft {#manual-agent}

Для проверки нужен принимающий HTTP-сервер, доступный из процесса WebSoft.
Публичный материал не включает тестовый сервер: используйте свой echo-сервис
или тестовый endpoint интеграции и замените `apiBase` и учебные маршруты
`/echo`, `/status`, `/binary`, `/gzip`, `/delay`, `/redirect` на его адреса.

Создайте обычный серверный агент WebSoft. В начало кода поместите входные
переменные из раздела «Как пользоваться». Для рецептов с файлами заранее
положите контрольный файл в файловую систему сервера и задайте `inputFile`.
Для proxy укажите доступные из WebSoft `proxyHost` и `proxyPort`.

```js
var apiBase = 'http://http-peer:18991';
var is434 = false; // Флаг совместимости: true для 434, false для 906+.
var token = 'test-token';
var username = 'demo';
var password = 'p+a&ss';
var inputFile = '/WebsoftServer/http-recipe-binary.bin';
var outputFile = '/tmp/http-recipe-download.bin';
var proxyHost = 'http-peer';
var proxyPort = 18991;
```

Затем вставьте код выбранного рецепта. Для первого запуска удобен
[GET с параметрами](#get-query): ему не нужны токен и файлы. В конец агента
добавьте временный вывод результата:

```js
EnableLog('http_recipe', true);
LogEvent('http_recipe', EncodeJson(result));
```

После запуска агента откройте именованный журнал `http_recipe`. Для echo-сервиса
в нём должен быть фактически полученный метод, адрес, параметры, заголовки и
тело. Исключение до записи в журнал появится как ошибка выполнения агента.

В учебных рецептах используются только фиктивные учётные данные. При проверке
боевого API не записывайте весь `result`: echo-ответ содержит `Authorization`,
а обычный ответ может содержать персональные данные. Логируйте только статус,
длительность и очищенное описание ошибки.

После проверки удалите контрольные файлы и временный вывод, если они больше
не нужны.

## Выбрать рецепт

| Задача | Рецепт | Сборки |
| --- | --- | --- |
| Поиск с кириллицей и специальными символами | [GET с параметрами](#get-query) | 434, 906, 1132, 1333, 1525 |
| Создать объект, передать Bearer и JSON | [POST JSON](#json-post) | 434, 906, 1132, 1333, 1525 |
| Передать форму и Basic | [Форма](#form-basic) | 434, 906, 1132, 1333, 1525 |
| Изменить или удалить объект | [PUT, PATCH, DELETE](#methods) | 434, 906, 1132, 1333, 1525 |
| Разделить 2xx, пустое тело и ошибки API | [Статусы](#status-handling) | 434, 906, 1132, 1333, 1525 |
| Ограничить ожидание | [Таймаут](#timeout) | 906+ |
| Отправить пачку без накопления заголовков | [Серия](#batch) | 434, 906, 1132, 1333, 1525 |
| Прочитать сжатый JSON | [Gzip](#gzip) | 434, 906, 1132, 1333, 1525 |
| Сохранить бинарный ответ | [Скачивание](#download) | 434, 906, 1132, 1333, 1525 |
| Передать файл как тело | [Бинарная загрузка](#upload) | 434, 906, 1132, 1333, 1525; встроенный вызов |
| Передать форму с одним файлом | [Встроенный multipart](#multipart-native) | 434, 906, 1132, 1333, 1525; ограничения ниже |
| Передать файл через DLL | [Multipart DLL](#multipart-dll) | 1333, 1525 |
| Отправить запрос через HTTP proxy | [Proxy](#proxy) | 434, 906, 1132, 1333, 1525 |
| Получить 302 без перехода | [Редирект](#redirect) | 434, 906, 1132, 1333, 1525 |

## GET: поиск с параметрами {#get-query}

Практический случай: поиск сотрудника по строке с пробелом, `+`, `&` и кириллицей.
`UrlEncodeQuery` принимает объект: кодируйте значения через него, не собирайте
строку запроса вручную. Peer должен получить исходную строку целиком.

<!-- recipe: get-query -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var query = UrlEncodeQuery({search: 'Иван Петров + R&D', page: 2});
var url = apiBase + '/echo?' + query;
try {
    if (is434) response = http.Open(url, 'get', '', '', handler);
    else response = http.Open(url, 'get', '', '', handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## POST: JSON и Bearer {#json-post}

Практический случай: создать заявку. Учебный endpoint возвращает `201 Created`.
Сериализация выполняется один раз; `application/json` без charset работает
на всех пяти DLL. Проверяются вложенный объект, массив, число и кириллица.

<!-- recipe: json-post -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/status?status=201';
var payload = {title: 'Заявка № 7', employee: {id: 'E-17'}, tags: ['a+b', 'R&D'], active: true, count: 2};
var headers = 'Content-Type: application/json\nAuthorization: Bearer ' + token + '\n';
try {
    if (is434) response = http.Open(url, 'post', EncodeJson(payload), headers, handler);
    else response = http.Open(url, 'post', EncodeJson(payload), headers, handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = {status: response.RespCode, data: ParseJson(response.Body)};
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## POST: форма и Basic {#form-basic}

Практический случай: старый сервис принимает `application/x-www-form-urlencoded`
и Basic. JSON и form — разные контракты. Пример проверяет, что `+`, `&` и
кириллица пережили кодирование формы, а заголовок Basic содержит пару login:password.

<!-- recipe: form-basic -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/echo';
var body = UrlEncodeQuery({name: 'Иван Петров', value: 'a+b&c=1'});
var headers = 'Content-Type: application/x-www-form-urlencoded\n' +
    'Authorization: Basic ' + Base64Encode(username + ':' + password) + '\n';
try {
    if (is434) response = http.Open(url, 'post', body, headers, handler);
    else response = http.Open(url, 'post', body, headers, handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## PUT, PATCH, DELETE {#methods}

Практический случай: обновить объект полностью, изменить отдельные поля,
удалить его. В учебном примере все три запроса идут на echo, который позволяет
проверить фактический метод и тело. В интеграции выбирайте один нужный вызов;
успешный DELETE может возвращать 204, как в следующем рецепте.

<!-- recipe: methods -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var methods = ['put', 'patch', 'delete'];
result = new Array();
var body = null;
var headers = null;
try {
    for (var i = 0; i < ArrayCount(methods); i++) {
        response = null;
        body = '';
        headers = '';
        if (methods[i] != 'delete') {
            body = EncodeJson({name: 'Новое имя'});
            headers = 'Content-Type: application/json\n';
        }
        try {
            if (is434) response = http.Open(apiBase + '/echo', methods[i], body, headers, handler);
            else response = http.Open(apiBase + '/echo', methods[i], body, headers, handler, null, 15);
            if (response == undefined || response == null)
                throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
            if (response.RespCode < 200 || response.RespCode >= 300)
                throw 'HTTP ' + response.RespCode;
            result.push(ParseJson(response.Body));
        } catch (errRequest) {
            throw errRequest;
        } finally {
            if (response != undefined && response != null) response.Dispose();
            response = null;
        }
    }
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## HTTP-статус, пустое тело и ошибка API {#status-handling}

DLL возвращает объект и для 4xx/5xx. Проверяйте статус самостоятельно, а
`ParseJson` вызывайте только для непустого JSON. `204` должен дать `data = null`.
Сетевой сбой обрабатывается отдельно через пустой результат и `ErrorMessage`.
Если API может вернуть HTML вместо JSON, перехватите ошибку разбора, сохранив
статус для диагностики. 429/500 не означают, что POST безопасно повторять.

<!-- recipe: status-handling -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var statuses = [200, 201, 204, 400, 401, 429, 500];
result = new Array();
var url = null;
var status = null;
var text = null;
var data = null;
var outcome = null;
try {
    for (var i = 0; i < ArrayCount(statuses); i++) {
        response = null;
        try {
            url = apiBase + '/status?status=' + statuses[i];
            if (is434) response = http.Open(url, 'get', '', '', handler);
            else response = http.Open(url, 'get', '', '', handler, null, 15);
            if (response == undefined || response == null)
                throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
            status = response.RespCode;
            text = response.Body;
            data = null;
            // Контракт этого API: непустые ответы, включая ошибки, содержат JSON.
            if (text != '') data = ParseJson(text);
            outcome = 'api-error';
            if (status >= 200 && status < 300) outcome = 'success';
            result.push({status: status, outcome: outcome, data: data});
        } catch (errRequest) {
            throw errRequest;
        } finally {
            if (response != undefined && response != null) response.Dispose();
            response = null;
        }
    }
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Таймаут: не повторять операцию вслепую {#timeout}

**Только 906, 1132, 1333, 1525.** Локальный сервис ждёт две секунды, а клиент —
не больше одной. Ожидается транспортная ошибка, а не HTTP-статус. Сервер может
успеть принять и выполнить запрос до таймаута; повтор POST требует отдельного
контракта идемпотентности. На 434 тест честно пропускается: аргумент отсутствует.

<!-- recipe: timeout -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
if (is434) throw 'Timeout argument is unavailable in build 434';
try {
    response = http.Open(apiBase + '/delay?ms=2000', 'get', '', '', handler, null, 1);
    if (response == undefined || response == null) {
        result = {outcome: 'transport-error', code: http.Error, message: http.ErrorMessage};
    } else {
        result = {outcome: 'response', status: response.RespCode};
    }
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Серия запросов с одним соединением {#batch}

Практический случай: отправить небольшую пачку. Общий handler сохраняет пул
соединений; новый внутренний клиент каждого `Open` не накапливает заголовки
предыдущих запросов. `client = null` также позволяет задавать таймаут каждому
вызову на 906+. В тесте три запроса должны пройти через одно соединение,
с `X-Sequence: 0`, `1`, `2` и без cookies. Это последовательная серия, не
пример фонового пула для параллельных агентов.

<!-- recipe: batch -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
result = new Array();
var headers = null;
try {
    // Для независимых REST-запросов не сохраняем cookie-сессию сервера.
    handler.UseCookies = false;
    for (var i = 0; i < 3; i++) {
        response = null;
        try {
            headers = 'Authorization: Bearer ' + token + '\nX-Sequence: ' + i + '\n';
            if (is434) response = http.Open(apiBase + '/echo', 'get', '', headers, handler);
            else response = http.Open(apiBase + '/echo', 'get', '', headers, handler, null, 15);
            if (response == undefined || response == null)
                throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
            if (response.RespCode < 200 || response.RespCode >= 300)
                throw 'HTTP ' + response.RespCode;
            result.push(ParseJson(response.Body));
        } catch (errRequest) {
            throw errRequest;
        } finally {
            if (response != undefined && response != null) response.Dispose();
            response = null;
        }
    }
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Получить gzip JSON {#gzip}

Практический случай: API отвечает с `Content-Encoding: gzip`. Распаковка
настраивается до первого запроса. Для моста SP-XML/.NET используется значение
типа `DecompressionMethods`, а не присваивание числа свойству enum.

<!-- recipe: gzip -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/gzip';
try {
    var systemAssembly = tools.dotnet_host.Object.GetAssembly('System.Private.CoreLib.dll');
    var enumType = systemAssembly.CallClassStaticMethod(
        'System.Type', 'GetType', ['System.Net.DecompressionMethods, System.Net.Primitives']
    );
    handler.AutomaticDecompression = systemAssembly.CallClassStaticMethod(
        'System.Enum', 'ToObject', [enumType, 3]
    );
    if (is434) response = http.Open(url, 'get', '', '', handler);
    else response = http.Open(url, 'get', '', '', handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

Значение `3` включает GZip и Deflate; этот рецепт проверяет именно gzip.

## Скачать бинарный файл через DLL {#download}

Практический случай: выгрузка отчёта или вложения. Сохраняйте `BinaryBody`,
не преобразуя файл в текст. Учебный файл содержит нулевой байт и байты выше 127;
его Base64 после сохранения должен быть `AAF/gP8NCg==`. `outputFile` — путь
в файловой системе сервера, не компьютера пользователя; он будет перезаписан.

<!-- recipe: download -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/binary';
try {
    if (is434) response = http.Open(url, 'get', '', '', handler);
    else response = http.Open(url, 'get', '', '', handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    PutFileData(outputFile, response.BinaryBody);
    result = {status: response.RespCode, savedBase64: Base64Encode(LoadFileData(outputFile))};
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Отправить файл как бинарное тело {#upload}

Практический случай: endpoint принимает `application/octet-stream` без multipart.
У встроенного вызова сохраняются байты `LoadFileData`; DLL перекодирует строковый
body в UTF-8 и для произвольного файла не подходит. Последний заголовок тоже
завершается `\n`. В наших Linux-конфигурациях встроенный HTTPS принял недоверенный
сертификат: этот рецепт проверяет байты по локальному HTTP и не подтверждает
безопасность встроенного TLS для внешней интеграции.

<!-- recipe: upload -->
```js
var response = HttpRequest(
    apiBase + '/echo', 'post', LoadFileData(inputFile),
    'Content-Type: application/octet-stream\nIgnore-Errors: 1\n'
);
if (response.RespCode < 200 || response.RespCode >= 300)
    throw 'HTTP ' + response.RespCode;
var result = ParseJson(response.Body);
```

## Multipart: фиксированная форма с одним файлом {#multipart-native}

Практический случай: поле `note` и файл `sample.bin`. Работает на всех пяти
сборках, но имеет узкую область применения: имена полей и файла фиксированы,
а содержимое не должно включать заменяемую строку `name="file"`. Это не общий
сериализатор для произвольных вложений. Для такого случая нужен адаптер,
который формирует MIME без подмены строк внутри файла. Ограничение встроенного
TLS из предыдущего рецепта сохраняется.

<!-- recipe: multipart-native -->
```js
var body = MultipartFormEncode('note', 'Привет', 'file', LoadFileData(inputFile));
var separator = body.indexOf('\r\n\r\n');
if (separator < 0) throw 'Multipart header separator was not found';
var contentHeader = StrLeftRange(body, separator);
body = StrRightRangePos(body, separator + 4);
body = StrReplace(body, 'name="file"',
    'name="file"; filename="sample.bin"\r\nContent-Type: application/octet-stream');
var response = HttpRequest(
    apiBase + '/echo', 'post', body, contentHeader + '\nIgnore-Errors: 1\n'
);
if (response.RespCode < 200 || response.RespCode >= 300)
    throw 'HTTP ' + response.RespCode;
var result = ParseJson(response.Body);
```

## Multipart через DLL: POST в 1333 и 1525 {#multipart-dll}

**Только 1333 и 1525.** Восьмой аргумент задаёт части формы. DLL сама выставляет
boundary и Content-Type. Это только POST: реализация игнорирует другой method
в этой ветви. В исходниках не освобождаются явно потоки файлов запроса;
единичный успешный тест не подтверждает пригодность для массовой отправки.
Для массовой передачи предпочтителен адаптер с корректным освобождением потоков.

<!-- recipe: multipart-dll -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var parts = [
    {Type: 'field', FieldName: 'note', Value: 'Привет'},
    {Type: 'file', FieldName: 'file', FilePath: inputFile,
        FileName: 'sample.bin', ContentType: 'application/octet-stream'}
];
try {
    response = http.Open(apiBase + '/echo', 'post', '',
        'Authorization: Bearer ' + token + '\n', handler, null, 15, EncodeJson(parts));
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## HTTP proxy с явным адресом {#proxy}

Практический случай: исходящие запросы должны идти через proxy. В этом тесте
`apiBase` использует зарезервированный адрес `http://http-target.invalid`, а peer
принимает запрос как HTTP proxy. Проверяется абсолютный request target;
внешней пересылки нет. HTTPS CONNECT, proxy-auth, NTLM и Kerberos этим рецептом
не проверяются.

При ручном запуске из агента замените заданный в начале страницы `apiBase`:

```js
var apiBase = 'http://http-target.invalid';
```

Это адрес назначения запроса; `proxyHost = 'http-peer'` остаётся адресом
локального тестового proxy. В рабочей интеграции укажите настоящий адрес API
и параметры вашего proxy.

<!-- recipe: proxy -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/echo';
try {
    var proxyUri = tools.dotnet_host.Object.GetAssembly('System.Private.Uri.dll')
        .CreateClassObject('System.UriBuilder');
    proxyUri.Scheme = 'http';
    proxyUri.Host = proxyHost;
    proxyUri.Port = proxyPort;
    var proxyObject = tools.dotnet_host.Object.GetAssembly('System.Net.WebProxy.dll')
        .CreateClassObject('System.Net.WebProxy');
    proxyObject.Address = proxyUri.Uri;
    handler.Proxy = proxyObject;
    handler.UseProxy = true;
    if (is434) response = http.Open(url, 'get', '', '', handler);
    else response = http.Open(url, 'get', '', '', handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    if (response.RespCode < 200 || response.RespCode >= 300)
        throw 'HTTP ' + response.RespCode;
    result = ParseJson(response.Body);
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Получить 302 без автоматического перехода {#redirect}

Практический случай: API неожиданно перенаправляет запрос на страницу входа.
Отключите редирект до первого запроса, прочитайте статус и Location.
Не пересылайте Bearer на адрес из Location автоматически: сначала проверьте
допустимый origin и контракт интеграции.

<!-- recipe: redirect -->
```js
var http = tools.dotnet_host.Object.GetAssembly('Websoft.HttpRequest.dll')
    .CreateClassObject('Websoft.HttpRequest.HttpRequest');
var handler = http.CreateNewHttpClientHandler(false);
var response = null;
var result = null;
var url = apiBase + '/redirect?status=302';
try {
    handler.AllowAutoRedirect = false;
    if (is434) response = http.Open(url, 'get', '', '', handler);
    else response = http.Open(url, 'get', '', '', handler, null, 15);
    if (response == undefined || response == null)
        throw 'HTTP transport ' + http.Error + ': ' + http.ErrorMessage;
    result = {status: response.RespCode, headers: ParseJson(response.Headers)};
} catch (err) {
    throw err;
} finally {
    if (response != undefined && response != null) response.Dispose();
    handler.Dispose();
}
```

## Как проверялись рецепты {#local-run}

Внутренний runner извлекал отмеченные блоки кода прямо из этой страницы без
изменений и выполнял весь применимый набор по очереди на сборках 434, 906, 1132,
1333 и 1525. На каждой сборке рецепты запускались отдельно с MSSQL и PostgreSQL.
Временный обработчик принимал только фиксированный идентификатор рецепта и был
защищён случайным ключом; произвольный код и адрес из запроса не принимались.

Отчёт сохранял исходную страницу, точный обработчик, SHA-256 каждого рецепта,
ответ SP-XML и независимую запись принимающей стороны. Проверяется не только
HTTP 200: для файлов сравниваются байты, для JSON — структура, для формы —
декодированные поля, для серии — соединения и заголовки. `SKIP` означает отсутствие
нужной сигнатуры DLL, а `FAIL` — ошибку рецепта, инфраструктуры или расхождение
с ожидаемым результатом. Пропуски не считаются успешными проверками.

<!-- recipe-results:start -->
Проверено по сохранённым прогонам UTC 2026-09-20. Каждый результат подтверждён на MSSQL и PostgreSQL; расхождений не обнаружено.

| Рецепт | 434 | 906 | 1132 | 1333 | 1525 |
| --- | --- | --- | --- | --- | --- |
| get-query | PASS | PASS | PASS | PASS | PASS |
| json-post | PASS | PASS | PASS | PASS | PASS |
| form-basic | PASS | PASS | PASS | PASS | PASS |
| methods | PASS | PASS | PASS | PASS | PASS |
| status-handling | PASS | PASS | PASS | PASS | PASS |
| timeout | SKIP | PASS | PASS | PASS | PASS |
| batch | PASS | PASS | PASS | PASS | PASS |
| gzip | PASS | PASS | PASS | PASS | PASS |
| download | PASS | PASS | PASS | PASS | PASS |
| upload | PASS | PASS | PASS | PASS | PASS |
| multipart-native | PASS | PASS | PASS | PASS | PASS |
| multipart-dll | SKIP | SKIP | SKIP | PASS | PASS |
| proxy | PASS | PASS | PASS | PASS | PASS |
| redirect | PASS | PASS | PASS | PASS | PASS |

**66 PASS, 4 SKIP, 0 FAIL** по матрице сборок.
`SKIP`: таймаут в 434 и multipart DLL до 1333. Исходные отчёты и хеши
сохраняются локально в `references/http-request/recipes/`; проприетарные файлы
и сырые результаты не публикуются.
<!-- recipe-results:end -->

Проверка относится к локальным Linux-контейнерам. Наличие двух БД проверяет
исполнение в обеих конфигурациях, но не превращает HTTP-рецепты в тесты SQL.
Windows, боевые API, доверенные корпоративные CA, mTLS и нагрузка в тысячи
запросов не входят в этот набор.
