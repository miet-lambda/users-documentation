# Документация по написанию Lua-скриптов для HTTP-сервиса

## Введение
Сервис позволяет выполнять пользовательские Lua-скрипты для обработки HTTP-запросов. Скрипты имеют доступ к:
- Данным входящего запроса.
- Формированию ответа.
- HTTP-клиенту для внешних запросов.
- Хранилищу ключ-значение (KV Storage).
- JSON-сериализации.

---

## Доступные функции

### 1. Работа с HTTP-контекстом
```lua
local context = require('miet.http.context').get()

-- Входящий запрос
local request = context:request()
local method = request['method']  -- GET, POST и т.д.
local url = request['url']        -- URL запроса
local query = request['query']    -- Параметры запроса (таблица)
local headers = request['headers']-- Заголовки (таблица)
local body = request['body']      -- Тело запроса (строка)

-- Исходящий ответ
local response = context:response()
response['status'] = 201          -- Установка HTTP-статуса
response['headers'] = {           -- Заголовки ответа
  ['Content-Type'] = 'application/json'
}
response['body'] = {              -- Тело ответа (строка/таблица)
  message = 'Hello'
}
```
## 2. Работа с HTTP клиентом
```lua
local client = require('miet.http.client').get()

-- Отправка запроса
local response, err = client:send('POST', 'https://api.example.com', {
  query = { key = 'value' },      -- Параметры URL
  headers = {                     -- Заголовки
    ['X-Auth'] = 'token'
  },
  body = { data = 'test' }        -- Тело (строка/таблица)
})

-- Сокращенные методы
client:get(url, params)
client:post(url, params)
client:head(url, params)
client:options(url, params)
-- И др.
```

## 3. Хранилище ключ-значение (KV Storage)
В рамках одного проекта у вас есть доступ к key-value хранлищу
```lua
local storage = require('miet.kv.storage').get()

-- Сохранение данных
storage:store('name', 'Alice')    -- Строка
storage:store('age', 25.5)        -- Число
storage:store('active', true)     -- Булево
storage:store('data', {           -- Таблица (сериализуется в JSON)
  items = { 'a', 'b' }
})

-- Получение данных
local name = storage:get('name'):as_string()
local age = storage:get('age'):as_number()
local active = storage:get('active'):as_boolean()
local data = storage:get('data'):as_table() -- JSON → Lua-таблица
```
## 4. Работа с JSON
```lua
local json = require('dkjson')

-- Кодирование
local data = { name = 'Alex', age = 30 }
local json_text = json.encode(data, { indent = true })

-- Декодирование
local decoded_data, pos, err = json.decode(json_text)
if err then
  -- Обработка ошибки
end
```
# Запрещенные функции
## Следующие функции недоступны из соображений безопасности:

```lua
os                    -- Запуск системных команд
io                    -- Доступ к файловой системе
package               -- Загрузка внешних библиотек
dofile()              -- Чтение локальных файлов
getfenv()             -- Доступ к окружению
debug.getregistry()   -- Доступ к внутренним структурам Lua
print()               -- Вывод в stdout
```

# Примеры использования
## Пример 1: Валидация запроса
```lua
local context = require('miet.http.context').get()
local request = context:request()
local response = context:response()

if request['method'] ~= 'POST' then
  response['status'] = 405
end

local data = json.decode(request['body'])
if not data.email or not data.password then
  response['status'] = 400
end
```
## Пример 2: Запрос к внешнему API
```lua
local client = require('miet.http.client').get()
local response, err = client:post('https://auth-service.com/login', {
  body = {
    username = 'user',
    password = 'secret'
  }
})

if err then
  -- Обработка ошибки
end

local token = json.decode(response['body']).token
```
## Пример 3: Кэширование данных
```lua
local storage = require('miet.kv.storage').get()
local cached_data = storage:get('cached_data'):as_table()

if not cached_data then
  -- Загрузка данных, если их нет в кэше
  storage:store('cached_data', cached_data)
end

return cached_data
```
# Ограничения ресурсов
## Таймаут выполнения: Скрипт прерывается, если превышает лимит времени (По умолчанию 5 секунд).

## Память: Превышение лимита памяти приводит к ошибке (По умолчанию лимит 10 MiB).
