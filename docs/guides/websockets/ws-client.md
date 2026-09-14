# Тестовый HTML-клиент

HTML-клиент для примеров руководства сгенерирован ИИ. Скопируйте код целиком в настраиваемый шаблон по инструкции для [сборки 434](434/connection.md) или [сборок 906, 1132, 1333 и 1525](906/connection.md).

На этой странице приведён только исходный код: соединение с сервером здесь не открывается.

Кнопка проверки связи и таймер подходят для сборок 906, 1132, 1333 и 1525:
клиент отправляет `$hbt`, а сервер отвечает `$ok`. Это не WebSocket
Ping/Pong-кадры, а служебные текстовые сообщения Datex.XHTTP. Автоматическая
проверка изначально выключена.

В сборке 434 не используйте эту функцию: `$hbt` попадёт в `main_ws_service` как обычная строка и вызовет ошибку `ParseJson()`.

В поле сообщения вводите JSON-команды `main_ws_service`. Полученные ответы клиент показывает как есть, включая массивы JSON.

## Код ws-client.html

```html
<!DOCTYPE html>
<html lang="ru">

<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>WebSocket Test Client</title>
    <style>
        /* базовые стили контролов (без оформления body/html, чтобы не конфликтовать с Bitrix) */
        input,
        textarea,
        select,
        button {
            font-family: inherit;
        }

        input[type="text"],
        textarea,
        select,
        input[type="number"] {
            width: 100%;
            border: 1px solid #ccc;
            border-radius: 6px;
            padding: 8px 10px;
            font-size: 14px;
            box-sizing: border-box;
        }

        input:focus,
        textarea:focus,
        select:focus {
            outline: none;
            border-color: #2684ff;
            box-shadow: 0 0 0 3px rgba(38, 132, 255, 0.2);
        }

        button {
            border: 1px solid #999;
            border-radius: 6px;
            padding: 8px 14px;
            font-size: 14px;
            cursor: pointer;
            background: #f2f2f2;
            transition: background 0.2s, border-color 0.2s;
        }

        button:hover {
            background: #e6e6e6;
        }

        button.primary {
            background: #2684ff;
            color: white;
            border-color: #0f5ad3;
        }

        button.primary:hover {
            background: #0f5ad3;
        }

        button.danger {
            background: #e74c3c;
            color: white;
            border-color: #c0392b;
        }

        button.danger:hover {
            background: #c0392b;
        }

        button:disabled {
            opacity: 0.6;
            cursor: not-allowed;
        }

        .panel {
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 1rem;
            margin-bottom: 1rem;
            background: #fff;
        }

        .controls {
            display: flex;
            gap: .5rem;
            flex-wrap: wrap;
            margin-top: .5rem;
        }

        .row {
            display: flex;
            gap: .5rem;
            align-items: center;
            flex-wrap: wrap;
            margin-top: .5rem;
        }

        .muted {
            opacity: .75;
        }

        #log {
            font-family: monospace;
            background: #f9f9f9;
            border: 1px solid #ccc;
            border-radius: 6px;
            padding: .5rem;
            height: 300px;
            overflow-y: auto;
            white-space: pre-wrap;
        }

        /* заголовок со статусом справа */
        .header {
            display: flex;
            align-items: center;
            justify-content: flex-start;
            gap: 12px;
            margin-bottom: 8px;
        }

        .header h2 {
            margin: 0;
        }

        /* бейдж статуса */
        .status {
            display: inline-flex;
            align-items: center;
            gap: 8px;
            padding: 6px 10px;
            border-radius: 6px;
            border: 1px solid #ccc;
            background: #f9f9f9;
            font-weight: 600;
            font-size: 14px;
        }

        .status .dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background: #999;
            display: inline-block;
        }

        .status.connected .dot {
            background: #2ecc71;
        }

        .status.connecting .dot {
            background: #f1c40f;
        }

        .status.disconnected .dot {
            background: #e74c3c;
        }
    </style>
</head>

<body>
    <div class="header">
        <h2>WebSocket Test Client</h2>
        <div id="status" class="status disconnected" title="Статус соединения">
            <span class="dot"></span>
            <span id="state">DISCONNECTED</span>
        </div>
    </div>

    <section class="panel">
        <label>URL WebSocket:</label>
        <input id="url" type="text" value="ws://localhost:80/services/main_ws_service?X-StatefulSocketId=123" />

        <label style="margin-top:8px;">Субпротоколы (через запятую):</label>
        <input id="protocols" type="text" placeholder="Например: json, chat, mqtt" />

        <div class="controls">
            <button id="connect" class="primary">Подключиться</button>
            <button id="disconnect" class="danger" disabled>Отключиться</button>
        </div>
    </section>

    <!-- Проверка связи Datex.XHTTP сборок 906, 1132, 1333 и 1525 -->
    <section class="panel">
        <label><input id="enablePing" type="checkbox" /> Проверять связь (906+)</label>
        <div class="row">
            <label for="pingInterval">Интервал (сек):</label>
            <input id="pingInterval" type="number" min="5" step="1" value="30" style="max-width:120px" />
            <button id="sendPing">Проверить связь</button>
        </div>
        <div class="row muted">
            <div>последний запрос: <span id="lastPing">—</span></div>
            <div>•</div>
            <div>последний ответ: <span id="lastPong">—</span></div>
        </div>
    </section>

    <section class="panel">
        <label>Сообщение:</label>
        <textarea id="payload" placeholder="Введите JSON-команду..."></textarea>
        <div class="controls">
            <button id="sendText" class="primary">Отправить</button>
            <button id="clear">Очистить лог</button>
        </div>
    </section>

    <section class="panel">
        <h3>Журнал</h3>
        <div id="log"></div>
    </section>

    <script>
        (() => {
            const url = document.getElementById('url');
            const protocols = document.getElementById('protocols');
            const connectBtn = document.getElementById('connect');
            const disconnectBtn = document.getElementById('disconnect');
            const payload = document.getElementById('payload');
            const sendText = document.getElementById('sendText');
            const clearBtn = document.getElementById('clear');
            const log = document.getElementById('log');
            const stateEl = document.getElementById('state');
            const statusBox = document.getElementById('status');

            // ping/pong UI
            const enablePing = document.getElementById('enablePing');
            const pingIntervalInput = document.getElementById('pingInterval');
            const sendPingBtn = document.getElementById('sendPing');
            const lastPingEl = document.getElementById('lastPing');
            const lastPongEl = document.getElementById('lastPong');

            let ws = null;
            let pingTimer = null;

            const now = () => new Date().toLocaleTimeString();

            function logMsg(msg, type = 'info') {
                const el = document.createElement('div');
                el.textContent = `[${now()}] ${msg}`;
                if (type === 'error') el.style.color = 'red';
                if (type === 'recv') el.style.color = 'blue';
                log.appendChild(el);
                log.scrollTop = log.scrollHeight;
                // console mirror
                if (type === 'error') console.error(msg); else console.log(msg);
            }

            function setState(s) {
                stateEl.textContent = s;
                statusBox.classList.remove('connected', 'connecting', 'disconnected');
                if (s === 'CONNECTED') statusBox.classList.add('connected');
                else if (s === 'CONNECTING') statusBox.classList.add('connecting');
                else statusBox.classList.add('disconnected');
            }

            function clearHeartbeat() {
                if (pingTimer) {
                    clearInterval(pingTimer);
                    pingTimer = null;
                }
            }

            function startHeartbeat() {
                clearHeartbeat();
                if (!enablePing.checked) return;
                let interval = Number(pingIntervalInput.value);
                if (!Number.isFinite(interval) || interval < 5) interval = 30;
                pingTimer = setInterval(() => {
                    sendPing();
                }, interval * 1000);
            }

            function sendPing() {
                if (!ws || ws.readyState !== WebSocket.OPEN) return;
                try {
                    // Служебный heartbeat Datex.XHTTP 1.24.4.27
                    ws.send('$hbt');
                    lastPingEl.textContent = now();
                    logMsg('→ $hbt');
                } catch (e) {
                    logMsg('Ошибка проверки связи: ' + e.message, 'error');
                }
            }

            function handleHeartbeat(data) {
                if (data !== '$ok') return false;
                lastPongEl.textContent = now();
                logMsg('← $ok');
                return true;
            }

            connectBtn.onclick = () => {
                if (ws) ws.close();
                const protoList = protocols.value.split(',').map(p => p.trim()).filter(Boolean);
                try {
                    ws = protoList.length ? new WebSocket(url.value, protoList) : new WebSocket(url.value);
                } catch (e) {
                    logMsg('Ошибка: ' + e.message, 'error');
                    return;
                }
                setState('CONNECTING');
                ws.onopen = () => {
                    setState('CONNECTED');
                    logMsg('Соединение установлено');
                    connectBtn.disabled = true;
                    disconnectBtn.disabled = false;
                    startHeartbeat();
                };
                ws.onmessage = e => {
                    // Ответ на heartbeat отметим отдельно, остальные сообщения покажем как есть
                    if (!handleHeartbeat(e.data)) {
                        logMsg('Получено: ' + e.data, 'recv');
                    }
                };
                ws.onerror = e => logMsg('Ошибка WebSocket', 'error');
                ws.onclose = e => {
                    setState('DISCONNECTED');
                    logMsg(`Закрыто: code=${e.code}, reason=${e.reason}`);
                    connectBtn.disabled = false;
                    disconnectBtn.disabled = true;
                    clearHeartbeat();
                };
            };

            disconnectBtn.onclick = () => { if (ws) ws.close(1000, 'manual'); };

            sendText.onclick = () => {
                if (!ws || ws.readyState !== WebSocket.OPEN) { logMsg('Нет активного соединения', 'error'); return; }
                ws.send(payload.value);
                logMsg('Отправлено: ' + payload.value);
            };

            clearBtn.onclick = () => { log.innerHTML = ''; };

            // Управление пингом
            enablePing.addEventListener('change', () => {
                if (ws && ws.readyState === WebSocket.OPEN) startHeartbeat();
            });
            pingIntervalInput.addEventListener('change', () => {
                if (ws && ws.readyState === WebSocket.OPEN) startHeartbeat();
            });
            sendPingBtn.addEventListener('click', sendPing);
        })();
    </script>
</body>

</html>
```
