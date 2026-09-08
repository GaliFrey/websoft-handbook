# WebSocket 434: подготовка и подключение

## Подготовка

Сначала войдите в WebTutor и откройте тестовый клиент в той же браузерной сессии, на том же сервере. `main_ws_service` проверяет доступ пользователя при подключении, а `main_socket` — при обработке сообщений. Если доступ не разрешён, одного правильного URL недостаточно.

Для примеров понадобится [тестовый HTML-клиент](../ws-client.md). Создайте настраиваемый шаблон и вставьте в него код клиента.

Пример ссылки на шаблон:

```
<протокол>://<адрес сервера>/_wt/doc_type/custom_web_template_id/<ID настраиваемого шаблона>
```

!!! warning "Важно"
    В рабочем окружении используйте защищённое соединение `wss` с настроенным TLS-сертификатом. На тестовом сервере без сертификата используйте `ws` и открывайте клиент по HTTP.

## Первое подключение

Откройте тестовый клиент, введите адрес сервиса в поле `URL WebSocket` и нажмите «Подключиться». При успешном подключении появится зелёный статус `🟢 CONNECTED`.

Пример адреса сервиса:

```
ws://localhost:80/services/main_ws_service
```

![Установленное WebSocket-соединение](../img/434/434-first-connect.png)

## Что отправлять серверу

Для команд `main_ws_service` нужен текстовый JSON с полями действия, например сообщение `init_socket` из [главы о группах](groups.md). Обычная строка `Привет` не является JSON и вызовет ошибку разбора. Не путайте это с отправкой текста **от сервера к клиенту** из агента.

```
15:39:50 [0164]	Invalid JSON.
Offset: 6
(ParseJson(),  x-local://wtv/wtv_main_ws_service.js,   line 66)
(x-local://wtv/wtv_main_ws_service.js,   line 66)
```

### Проверка связи

В сборке 434 служебная проверка связи через `$hbt` не поддерживается.

В Datex.XHTTP `1.22.6.9` отдельного обработчика `$hbt` нет: строка передаётся в `process()` сервиса, а `ParseJson("$hbt")` завершается ошибкой `Invalid format`.

```
15:41:53 [0251]	Invalid format
(ParseJson(),  x-local://wtv/wtv_main_ws_service.js,   line 66)
(x-local://wtv/wtv_main_ws_service.js,   line 66)
```

### Копия сервиса { #service-copy }

При запуске WebTutor читает файл `x-local://source/api_ext.xml` и регистрирует перечисленные в нём библиотеки. Используйте этот механизм, чтобы создать второй тестовый сервис:

1. Скопируйте `wtv_main_ws_service.xml` и `wtv_main_ws_service.js` из каталога `wtv` в каталог `source` под именами `rtk_ws_service.xml` и `rtk_ws_service.js`.
2. В `source/rtk_ws_service.xml` измените открывающий тег:

    ``` xml
    <SPXML-INLINE-FORM WEB-SERVICE-PATH="/services/rtk_ws_service" CODE-LIB="1">
    ```

    Атрибут `CODE-LIB="1"` подключает расположенный рядом файл с тем же базовым именем — `rtk_ws_service.js`.

3. Добавьте библиотеку в `source/api_ext.xml`. Если в файле уже есть другие элементы `api`, сохраните их и добавьте новый:

    ``` xml
    <api_ext>
        <apis>
            <api>
                <name>rtk_ws_service</name>
                <libs>
                    <lib>
                        <path>x-local://source/rtk_ws_service.xml</path>
                    </lib>
                </libs>
            </api>
        </apis>
    </api_ext>
    ```

4. При необходимости замените в `rtk_ws_service.js` диагностическую подпись `wtv_main_ws_service.js` на `rtk_ws_service.js`. На работу сервиса эта строка не влияет.
5. Перезапустите WebTutor, чтобы `api_ext.xml` был обработан при старте.
6. Откройте вторую копию тестового клиента и подключите её к `ws://localhost:80/services/rtk_ws_service`. Первую оставьте подключённой к `ws://localhost:80/services/main_ws_service`.

[Далее: отправка сообщений](messages.md).
