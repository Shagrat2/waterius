# Исправление: перезагрузка ESP при polling /api/status

**Дата:** 2026-03-30
**Ветка:** waterius2-new-settings

---

## Проблема

При настройке прибора на шаге "пролив воды" ESP8266 перезагружалась через некоторое время.
На этом шаге веб-интерфейс каждые 2 секунды опрашивает `/api/status/0` или `/api/status/1`,
чтобы показать количество импульсов. В итоге ESP уходила в перезагрузку.

## Причина

В JS (common.js) функции `getImpulses()`, `getImpulsesHall()`, `getStatus()` не проверяли,
завершился ли предыдущий HTTP-запрос, перед отправкой нового. При этом встроенная функция `ajax()`
содержала retry-логику (до 3 повторов через 1 сек при ошибке).

Если ESP отвечала медленно — параллельные запросы накапливались. Каждый запрос на стороне ESP
аллоцирует `AsyncResponseStream` через `beginResponseStream()` (~1680 байт на heap).
При ~30KB свободного heap несколько одновременных response-объектов быстро исчерпывали память.

**Было:**
```javascript
function getImpulses(i) {
    setTimeout(() => {
        ajax('/api/status/' + i, {}, data => {
            document.getElementById('impulses').textContent = data.impulses;
            formError(data.error);
            getImpulses(i);
        }, false);
    }, 2000);
}
```
Никакой защиты от параллельных запросов. `ajax()` внутри может породить retry.

## Что сделали

### 1. JS: защита от параллельных запросов (data/static/common.js)

Добавлена обёртка `_pollRequest()` (паттерн из проекта MeterRS485, `software/data/script.js`):

```javascript
var _pollPending = false;
var _pollAbortCtrl = null;
var _pollAbortTimer = null;

function _pollRequest(url, callback) {
    if (_pollPending) return;          // предыдущий не завершён — пропускаем
    _pollPending = true;
    _pollAbortCtrl = new AbortController();
    fetch(url, {signal: _pollAbortCtrl.signal})
        .then(res => res.ok ? res.json() : Promise.reject(res))
        .then(data => { _pollCleanup(); callback(data); })
        .catch(() => { _pollCleanup(); });
    _pollAbortTimer = setTimeout(() => {       // автоотмена через 3 сек
        if (_pollPending && _pollAbortCtrl) {
            _pollAbortCtrl.abort();
            _pollCleanup();
        }
    }, 3000);
}
```

Три механизма защиты:
- **Флаг `_pollPending`** — новый запрос не отправляется, пока предыдущий не завершён
- **`AbortController`** — позволяет отменить зависший fetch на стороне браузера
- **Таймаут 3 сек** — автоматический abort и сброс флага

Функции `getImpulses()`, `getImpulsesHall()`, `getStatus()`, `getWiFiStatus()` переведены на `_pollRequest()`.

### 2. C++: логирование heap для диагностики

Добавлены точки мониторинга:

| Место | Что логируется |
|---|---|
| `send_json_response()` | heap до и после `beginResponseStream` |
| `get_api_status()` | heap и фрагментация при каждом вызове |
| `get_api_main_status()` | аналогично |
| Цикл портала (каждые 5 сек) | free, frag%, max_block |

`max_block` — размер максимального свободного блока. Если `free` стабилен, но `max_block` падает — фрагментация.

### Затронутые файлы

- `data/static/common.js` — polling-логика (основное исправление)
- `src/portal/active_point_api.cpp` — логирование heap в API-хендлерах
- `src/portal/active_point.cpp` — логирование heap в цикле портала

## Результат

Тестирование: 8+ минут непрерывного polling `/api/status/1` на реальном устройстве.

| Метрика | Значение | Динамика |
|---|---|---|
| PORTAL HEAP free | 29776–29992 | стабильно ~30KB |
| PORTAL HEAP frag | 6–7% | стабильно |
| PORTAL HEAP max_block | 27360–28024 | стабильно |
| heap при входе в get_api_status | 27840–27928 | стабильно |
| Разница before/after response | ~1680 байт | стабильно возвращается |

- Polling работает строго по 1 запросу каждые ~2 секунды
- `beginResponseStream` ни разу не вернул NULL
- Перезагрузок ESP не зафиксировано
- Heap не деградирует со временем
