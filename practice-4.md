---
title: Tarantool App
tags: tarantool, trainings
description:
---

# Практика: Сервис сокращения ссылок

## ТЗ

- Пользователь хотел бы сократить ссылку:
    - Чтобы вместить её в смс или в твиттер
    - Сохранить в QR код
    - Или даже расположить в печатной продукции
- Необходимо хранение соответствий

## Общий план

- Настроить хранилище Tarantool
- Настроить репликацию
- Настроить масштабирование
- Обслуживание и диагностика
- Запустить http-сервер

## Часть 4 — сервер приложений

- Сегодня мы напишем легковесный `http` сервер
- Реализуем `http` `api` в своем приложении

## Intro

### Lua за 15 минут

- https://learnxinyminutes.com/docs/ru-ru/lua-ru/

### Код

- Код на языке `Lua` выполняется в файберах (их также называют корутины, горутины, зеленые треды, легковесные сопрограммы)
- Файберы работают по принципу кооперативной многозадачности
    - В текущий момент времени выполняется только один файбер, и этот файбер, если операция ввода-вывода или явно, передаёт управление другому файберу
- `Tarantool` содержит встроенные модули:
  - `log` — журналирование событий приложения и базы данных
  - `console` — предлагает возможность выполнения и отладки кода в уже запущенном инстансе тарантула
  - `clock` — функции для работы с системным временем
  - `digest` — вычисление хеша от строки
  - `fio` — работа с файлами
  - `socket` — работа с сетью
  - `box` — настройка и хранение данных
- Некоторыми из них вы уже пользовались
- Подробнее обо всех модулях: https://www.tarantool.io/en/doc/latest/reference/

## Сервер приложений

- Сделаем легковесный `http` сервер
- `http` сервер под капотом создаёт файбер, который
    - слушает `tcp/ip` `server:port`
    - для каждого нового клиента запускает `client_handler` в отдельном файбере
    - `client_handler` принимает первым аргументом соединение с клиентом
    - `client_handler` обрабывает запросы, вызывая обработчики роутов
- Функции обработки `http` запросов:
    - `GET /`
      вернет html форму для ввода ссылки
    - `GET /?url=*`
      сгенерирует и вернет короткую ссылку
    - `GET /*`
      перенаправит на исходную ссылку
- Доступ к данным будет выполняться всё также через `vshard.router.callrw`(`-ro`)

### Сервер обработки http запросов

- Возьмём топологию попроще для удобства запуска и отладки

- Установим фреймворк шардирования `vshard` и `http` сервер
    ```bash
    tarantoolctl rocks install vshard
    tarantoolctl rocks install http
    ```
- `topology.lua`
    ```lua []
    return {
        BUCKET_COUNT = 16,
        sharding = {
            ['aaaaaaaa-0000-0000-0000-000000000000'] = {
                -- replicaset #1
                replicas = {
                    ['aaaaaaaa-0000-0000-0000-000000000001'] = {
                        uri = 'guest@127.0.0.1:3301',
                        name = 'spb',
                        master = true
                    },
                },
            },
            ['bbbbbbbb-0000-0000-0000-000000000000'] = {
                -- replicaset #2
                replicas = {
                    ['bbbbbbbb-0000-0000-0000-000000000001'] = {
                        uri = 'guest@127.0.0.1:3401',
                        name = 'moscow',
                        master = true
                    },
                },
            },
        },
    }
    ```
- `schema.lua`
    ```lua 
    local schema = {}

    function schema.init()
        if box.info.ro then return end
        --[[
            откроем доступ без пароля
        ]]

        box.schema.user.grant(
            'guest', 'super', nil, nil,
            { if_not_exists=true }
        )

        --[[
            Создание схемы данных, индексов
        ]]

        --[[
            Таблица `redirector` — хранилище коротких ссылок
            if_not_exists — флаг, о том чтобы не генерировать исключение, если спейс уже существует
        ]]
        box.schema.space.create(
            'redirector',
            { if_not_exists=true }
        )

        box.space.redirector:format({
            { name='source', type='string' },
            { name='short', type='string' },
            { name='bucket_id', type='unsigned'},
        })

        --[[
            Первичный ключ, первый создаваемый индекс
        ]]
        box.space.redirector:create_index(
            'primary',
            {
                parts = { 'source' },
                if_not_exists = true,
            }
        )

        --[[
            Вторичный индекс, для быстрого поиска строки по короткой ссылке
        ]]
        box.space.redirector:create_index(
            'short',
            {
                parts = { 'short' },
                if_not_exists = true,
            }
        )

        --[[
            Индекс для шардирования данных
        ]]
        box.space.redirector:create_index(
            'bucket_id',
            {
                parts = { 'bucket_id' },
                unique = false,
                if_not_exists = true,
            }
        )
    end

    return schema
    ```
- `app.lua`
    ```lua
    local app = {}
    local digest = require 'digest'
    local vshard = require 'vshard'

    function app.url(short)
        local bucket_id = vshard.router.bucket_id_mpcrc32(short)
        local url, err = vshard.router.callro(
            bucket_id,
            'box.space.redirector.index.short:get',
            {short}
        )
        if url ~= nil then
            return url[1]
        elseif err ~= nil then
            error(err)
        else
            return
        end
    end

    function app.shorten(url)
        -- while true do
            --[[ map/reduce для поиска существующего url ]]
            local shards, err = vshard.router.routeall()
            if err ~= nil then error(err) end
            for uid, replica in pairs(shards) do
                local rec = replica:callro('box.space.redirector:get',{ url })
                if rec ~= nil then
                    return rec[2]
                end
            end

            --[[ пробуем создать ]]
            local short = digest.sha1_hex(url):sub(1,10)
            local bucket_id = vshard.router.bucket_id_mpcrc32(short)
            local success, err = vshard.router.callrw(
                bucket_id,
                'box.space.redirector:insert',
                { {url, short, bucket_id} },
                { timeout = 1 }
            )
            if success then
                return short
            elseif err ~= nil then
                error(err)
            end
        -- end
    end

    return app
    ```
- `server.lua`
    ```lua
    local log = require('log')
    local socket = require('socket')
    local digest = require('digest')
    local httpd = require('http.server').new('0.0.0.0', tonumber(os.getenv('TT_HTTP_PORT')))

    --[[
        HTTP сервер обработки запросов
    ]]
    local hostname = 'localhost' ---<<<!!!!! Вставить dns имя или ip своей виртуалки


    --[[
        HTML форма для создания короткой ссылки
    ]]
    local form = [[
    <html>
    <head></head>
    <body>
        <form>
        <input name=url type=text />
        <input type=submit />
        </form>
    </body>
    </html>
    ]]

    function root(req)
        local url = req:param('url')
        if url ~= nil then
            local short = app.shorten(url)
            if short == nil then
                return {
                    status = 404,
                }
            end

            local pasteable_short = 'http://' .. hostname .. ':' .. os.getenv('TT_HTTP_PORT') .. '/' .. short
            return {
                status = 200,
                headers = { ['content-type'] = 'text/html; charset=utf8' },
                body = pasteable_short
            }
        end

        return {
            status = 200,
            headers = { ['content-type'] = 'text/html; charset=utf8' },
            body = form
        }
    end

    function short(req)
        local short = req:stash('short')
        local source = app.url(short)
        if source == nil then
            return {status=404}
        end

        return {
            status = 301, 
            headers = { ['location'] = source },
        }
    end

    httpd:route(
        { path = '/:short' }, short)

        httpd:route(
        { path = '/' }, root)

    local function init()
        httpd:start()
    end

    local function stop()
        if httpd ~= nil then
            httpd:stop()
        end
        httpd = nil
    end

    return {
        init=init,
        stop=stop,
    }
    ```

- `init.lua`
    ```lua
    vshard = require('vshard')
    local topology = require('topology')
    local fio = require('fio')
    local schema = require('schema')

    local instance_name = assert(
        os.getenv('TT_INSTANCE_NAME'), "TT_INSTANCE_NAME required"
    )
    local data_dir = os.getenv('TT_DATADIR') or "data/"..instance_name
    if not fio.stat(data_dir) then
        fio.mktree(data_dir)
    end

    vshard.storage.cfg({
            bucket_count = topology.BUCKET_COUNT,
            sharding = topology.sharding,
            memtx_dir = data_dir,
            wal_dir = data_dir,
            replication_connect_quorum = 1,
        },
        assert(os.getenv('TT_INSTANCE_UUID',"TT_INSTANCE_UUID required"))
    )

    vshard.router.cfg{
        bucket_count = topology.BUCKET_COUNT,
        sharding = topology.sharding,
    }

    schema.init()
    app = require 'app'

    server = require 'server'
    server.init()
    ```
- `spb.sh`
    ```lua
    TT_INSTANCE_UUID=aaaaaaaa-0000-0000-0000-000000000001 \
    TT_INSTANCE_NAME=spb \
    TT_HTTP_PORT=8081 \
    tarantool init.lua
    ```
- `moscow.sh`
    ```lua
    TT_INSTANCE_UUID=bbbbbbbb-0000-0000-0000-000000000001 \
    TT_INSTANCE_NAME=moscow \
    TT_HTTP_PORT=8082 \
    tarantool init.lua
    ```

### Запуск

- Удалим устаревшие рабочие директории
    ```bash
    rm -rf data/*
    ```
- Запустим два узла
    ```bash
    ./spb.sh
    ```
    ```bash
    ./moscow.sh
    ```
- Вновь требуется бутстрап
    ```
    tarantoolctl connect 127.0.0.1:3301
    ```
    - `tarantool>`
      ```lua
      vshard.router.bootstrap()
      ```

## Оно заработало?

- Откроем браузер на `http://<ip_address>:8081/`
    - Должна появится форма для ввода `url`
- Введем любой `url`
- Получим короткую ссылку
- Перейдем по короткой ссылке
- Или можно взять `curl`
   ```bash
   curl -G "http://localhost:8081/"\
     --data-urlencode "url=https://mail.ru"
   ```
   ```bash
   curl -L $(!!)
   ```
- Или можно воспользоваться готовым скриптом, который пытается укоротить `https://google.com/?q=<random-string>`
- `test.lua`
    ```lua
    local log = require('log')
    local client = require('http.client')

    --[[
        Простой тест для проверки укорачивания ссылок
    ]]
    local host = '127.0.0.1'
    local port = 8081

    local source = 'https://google.com/?q=' .. tostring(os.time())
    local makeshort = 'http://' .. host .. ':' .. port .. '/?url='
    local response = client.get(makeshort .. source)
    local short = response.body
    print(short, '->', source)
    local redirect = client.get(short, {follow_location=false})

    assert(redirect.headers['location'] == source,
        "Source url and redirect not equal: " .. source .. " " ..
        redirect.headers['location'] )
    ```
- Запуск теста
    ```bash
    tarantool test.lua
    ```
- Примерный вывод
    ```bash
    http://localhost:8081/XniDtG2jrBa23Q
      ->    Https://google.com/?q=1607896232
    ```
