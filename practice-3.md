---
title: Tarantool Maentenance
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

## Часть 4 — обслуживание и диагностика

### Потоки `tarantool`

- `iproto`
  - Прием, передача сетевых сообщений
  - Используется в бинарных коннекторах к `Tarantool`
- `tx` (в том числе `luajit`)
  - Обработка данных lua скриптами
- `wal`
  - Запись транзакций в файл журнал

Запрос от клиента попадает в `iproto` поток. Затем передаётся в `tx`. В `tx` потоке потоке
обработка может создать транзакцию, которая при `box.commit()` передаётся в `wal` поток.
`wal` поток записывает транзакцию в файл и возвращает управление в `tx`. Затем
`tx` возвращает результат в `iproto` поток, который в свою очередь передаёт его клиенту.

Таким образом, строго говоря, обработка данных в Tarantool трехпоточна.

- Посмотреть потоки можно, например, утилитой `htop`.
    ``` bash
    tarantool
    
    tarantool> box.cfg{}
    ```

    ``` bash
    htop --tree
    # f2 (ctrl+f2, click "Setup")
    # Display options -> Show Custom Thread Names
    # f3 (ctrl+f3, click "Search") tarantool <enter>
    ```
- Примерный вывод
    ``` bash
    └─ tarantool
       ├─ coio
       ├─ coio
       ├─ wal
       └─ iproto
    ```

### Настройка сетевого потока

- readahead (байт) (min 128)
- net_msg_max (количество) (min 2)

#### readahead

``` bash
mkdir mnt1 && cd mnt1
```

- Сервер
    ``` bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        -- запускаем сервер
        box.cfg{listen=3301}
        -- для тестов даем права гостю
        box.schema.user.grant('guest', 'super', nil, nil, { if_not_exists = true })

        -- конфигурируем размер буфера чтения
        box.cfg{readahead=128}
        -- проверка
        box.cfg.readahead
        ```
- Генератор нагрузки
    ``` bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        netbox = require('net.box')
        c = netbox.connect('127.0.0.1:3301')
        fiber = require('fiber')

        -- создаем некоторое количество файберов для одновременных запросов
        for i=1,120 do
            fiber.new(c.eval, c, [[ a = ... ]], {('x'):rep(128)})
        end
        ```
- `stopping input on connection fd 17, aka 127.0.0.1:3301, peer of 127.0.0.1:48916, readahead limit is reached`


#### net_msg_max

- Почистим директорию от предыдущего запуска
    ``` bash
    rm -rf *.snap *.xlog
    ```

- Сервер
    ``` lua
    -- запускаем сервер
    box.cfg{listen=3301}
    -- Для тестов даем права гостю
    box.schema.user.grant('guest', 'super', nil, nil, { if_not_exists = true })

    -- конфигурируем максимальное количество одновременных сообщений
    box.cfg{net_msg_max=2}
    -- проверка
    box.cfg.net_msg_max
    ```
- Генератор нагрузки из другого инстанса
    ``` bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        netbox = require('net.box')
        c = netbox.connect('127.0.0.1:3301')

        fiber = require('fiber')
        -- создаем три одновременных запроса через файберы
        for i=1,3 do
            fiber.new(c.eval, c, [[a=...]], {('x'):rep(128)})
        end
        ```
- `stopping input on connection fd 17, aka 127.0.0.1:3301, peer of 127.0.0.1:48798, net_msg_max limit is reached`

### Настройка памяти

#### memtx_memory

- Почистим директорию от предыдущего запуска
    ``` bash
    rm -rf *.snap *.xlog
    ```

- Предельно минимальное значение для хранение данных в `memtx` 32mb
    ``` bash
    tarantool
    ```

    - `tarantool>`

        ``` lua
        -- выделим 32 мегабайта памяти
        box.cfg{memtx_memory = 32*1024*1024}

        -- создадим тестовый спейс
        box.schema.space.create('test', { if_not_exists=true })
        -- первичный ключ на нем
        box.space.test:create_index('primary', { if_not_exists=true })

        -- попытаемся вставить примерно 1 мегабайт информации
        box.space.test:put({1, ('x'):rep(1024*1024 - 30)})
        ```
- `error: Failed to allocate 1048567 bytes in slab allocator for memtx_tuple`

- Интроспекция слабов
```lua
local function MAP(t)
    return setmetatable(t, require "json".map_mt)
end
local t = require "fun".iter(box.slab.stats())
    :map(function(s)
        return MAP {
            -- kind: Какой максимальный размер тапла можно хранить
            kind = s.item_size,
            -- cnt: Сколько слабов под них выделено
            cnt = s.slab_count,
            -- items: Сколько уже таплов поселилось в слабах
            items = s.item_count,
            -- use: % занятости слаба
            use = math.floor(s.mem_used / (s.slab_size * s.slab_count) * 1000) / 10,
            -- size: Размер слаба в байтах (отношение slab_size/kind дает примерное количество таплов, которые поместятся в слаб)
            size = s.slab_size,
        }
    end)
    :totable()
table.sort(t, function(a, b) return a.kind < b.kind end)
return t
```

- То же самое в виде однострочника (`Ctrl+C -> Ctrl+V`)
``` lua
local function MAP(t) return setmetatable(t,require'json'.map_mt) end local t = require'fun'.iter(box.slab.stats()):map(function(s) return MAP{kind=s.item_size, cnt=s.slab_count, items=s.item_count, use=math.floor(s.mem_used/(s.slab_size * s.slab_count)*1000)/10, size=s.slab_size} end):totable() table.sort(t,function(a,b) return a.kind < b.kind end) return t
```

#### malloc

- Выделение памяти для таплов без спейса происходит в rss памяти отдельно от спейса
- Посмотрим на пределы
    ``` bash
    tarantool
    ```

    - `tarantool>`

        ``` lua
        _ = box.tuple.new{('x'):rep(512*1024*1000)}
        ```

- `error: Failed to allocate 32719 bytes in mpstream for reserve`

#### lua objects

``` bash
tarantool
```

- `tarantool>`
    ``` lua
    _ = ('x'):rep(1024*1024*1000)
    ```

- `error: not enough memory`

### Поток записи транзакций

- Почистим директорию от предыдущего запуска
    ``` bash
    rm -rf *.snap *.xlog
    ```

- Если коммит транзакции происходи слишком долго, необходима диагностика
- Например, если какой-то файбер надолго задержал управление
    ``` bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        box.cfg{}
        -- создадим тестовый спейс
        box.schema.space.create('test', { if_not_exists=true })
        -- первичный ключ на нем
        box.space.test:create_index('primary', { if_not_exists=true })

        function busyloop(secs)
            local clock = require 'clock'
            local d = clock.realtime()+secs
            while clock.realtime() < d do end
        end

        fiber = require('fiber')
        do
            fiber.new(busyloop, 2) -- шедулит файбер (не запускает)
            fiber.create(function() -- шедулит и запускает
                box.begin() -- отключает yield на операциях box'a
                box.space.test:put{1, 1}
                box.commit() -- делает fiber.yield, и ожидает успешной записи в wal
                -- если дошли сюда, то запись успешна
            end)
        end
        ```
- `too long WAL write: 1 rows at LSN 5: 2.009 sec`
- Если транзакции действительно большие и занимают много времени
  можно отредактировать `too_long_threshold`
    ```bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        box.cfg{too_long_threshold=10}
        -- создадим тестовый спейс
        box.schema.space.create('test', { if_not_exists=true })
        -- первичный ключ на нем
        box.space.test:create_index('primary', { if_not_exists=true })

        function busyloop(secs)
            local clock = require 'clock'
            local d = clock.realtime()+secs
            while clock.realtime() < d do end
        end

        fiber = require('fiber')
        do
            fiber.new(busyloop, 2)
            fiber.create(function()
                box.begin()
                box.space.test:put{1, 1}
                box.commit()
            end)
        end
        ```
- Сообщения не будет

### Повреждение файла `xlog` (`write ahead log`)

- Почистим директорию от предыдущего запуска
    ``` bash
    rm -rf *.snap *.xlog
    ```
- Запустим `tarantool`
    ```bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        box.cfg{}

        -- создадим тестовый спейс
        box.schema.space.create('test', { if_not_exists=true })
        -- первичный ключ на нем
        box.space.test:create_index('primary', { if_not_exists=true })

        -- сделаем одну транзакцию
        box.space.test:put({1, 'first data'})

        -- Ctrl+c
        ```

- Посмотреть транзакции можно модулем `xlog`

    ```bash
    tarantool
    ```

    - `tarantool>`
        ``` bash
        yaml = require('yaml')
        xlog = require('xlog')

        for _, transaction in xlog.pairs('00000000000000000000.xlog') do -- также для файлов .snap
            print(yaml.encode(transaction))
        end
        ```

- Повредим файл `xlog` с недавними транзакциями

    ``` bash
    printf '\x31\xc0\xc3' | dd of=00000000000000000000.xlog bs=1 seek=340 count=3 conv=notrunc
    ```

- Попытаемся запустить

    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` lua
        box.cfg{}
        ```

- `XlogError: tx checksum mismatch`

- Можно использовать `force_recovery`

    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` lua
        box.cfg{force_recovery=true}
        -- последней транзакции в спейсе не будет
        -- она была повреждена
        box.space.test:select{}
        ```

### Повреждение файла `snap` (`snapshot`)

- Почистим директорию от предыдущего запуска
    ``` bash
    rm -rf *.snap *.xlog
    ```
- Запустим `tarantool`
    ```bash
    tarantool
    ```
    - `tarantool>`
        ``` lua
        box.cfg{}

        -- создадим тестовый спейс
        box.schema.space.create('test', { if_not_exists=true })
        -- первичный ключ на нем
        box.space.test:create_index('primary', { if_not_exists=true })

        -- сделаем одну транзакцию
        box.space.test:put({1, 'first data'})

        -- сделаем snapshot данных
        box.snapshot()

        -- сделаем ещё одну транзакцию
        box.space.test:put({2, 'second data'})

        -- Ctrl+c
        ```

- Посмотреть снапшот можно тоже модулем `xlog`

    ```bash
    tarantool
    ```

    - `tarantool>`
        ``` bash
        yaml = require('yaml')
        xlog = require('xlog')

        for _, transaction in xlog.pairs('00000000000000000004.snap') do
            print(yaml.encode(transaction))
        end
        ```
    - Последняя транзакция `[1,"first data"]`

- Повредим файл snap с последней версий данных

    ``` bash
    printf '\x31\xc0\xc3' | dd of=00000000000000000004.snap bs=1 seek=340 count=3 conv=notrunc
    ```

- Попытаемся запустить

    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` lua
        box.cfg{}
        ```

- `XlogError: tx checksum mismatch`

- `force_recovery` тоже можно использовать для снапшотов

    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` lua
        box.cfg{force_recovery=true}
        ```

- Но давайте воспользуемся другим способом
- Проверим, что у нас есть предыдущий снапшот
    ```bash
    file 00000000000000000000.snap
    ```
- Удалим повреждённый снапшот
    ```bash
    mv 00000000000000000004.snap 00000000000000000004.snap.bak
    ```
- Запустим tarantool
    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` lua
        box.cfg{}

        box.space.test:select{}
        ```
    - `[1, 'first data']`
    - `[2, 'second data']`
- Запуск произошел с ближайшего снапшота и с применением всех последующих `xlog` файлов


### Вычисление размера спейса в оперативной памяти (Задача со зведочкой)

- Просуммировать размер спейса и размер всех его индексов

    ``` bash
    tarantool
    ```

    - `tarantool>`
        ``` bash
        _G.ls = setmetatable({}, {
          __serialize = function()
            local res = {}

            for _, sp_info in box.space._space:pairs(512, { iterator = "GE" }) do
              local sp   = box.space[sp_info.name]
              local info = {}
              info.name   = tostring(sp.name)
              info.engine = tostring(sp.engine)
              info.len    = tostring(sp:len())
              info.bsize  = sp:bsize()
              info.owner  = tostring(box.space._user:get(sp_info.owner).name)
              -- index numeration starts with zero
              if next(sp.index) then
                for k, v in pairs(sp.index) do
                    if type(k) == 'number' then
                        info.bsize = info.bsize + v:bsize()
                    end
                end
              end
              info.bsize = tostring(info.bsize):sub(1, -4) -- strip ULL
              table.insert(res, info)
            end
            table.sort(res, function(a, b) return a.engine .. a.name < b.engine .. b.name end)

            --do return res end

            local nice = {}
            local nice_l = {}
            for _, sp in pairs(res) do
              for k, v in pairs(sp) do
                if not nice_l[k] then
                  nice_l[k] = 0
                end

                nice_l[k] = math.max(nice_l[k], v:len())
              end
            end

            local split_size = tonumber(rawget(_G, '__LS_SPLIT_SIZE')) or 2
            for _, sp in pairs(res) do
              local str = ''
              for _, k in pairs({ 'engine', 'owner', 'bsize', 'len' }) do
                str = str .. ("%% -%ds%% %ds"):format(nice_l[k], split_size):format(sp[k], " ")
              end
              str = str .. sp['name']
              table.insert(nice, str)
            end

            return nice
          end;
        })
        ```
- Использование
  - `tarantool>`
    ```
    ls
    ```

### Ребутстрап репликации

- Запустим первый узел (Spb)
- `$`
```
mkdir spb
cd spb
tarantool
```
- `tarantool>`
```lua
box.cfg{listen='127.0.0.1:3301'}
box.schema.user.grant('guest', 'super')
box.schema.space.create('test')
box.space.test:create_index('pkey')

box.space.test:put{1, 'value1'}

```
- Запустим второй узел (Moscow)
- `$`
```
mkdir moscow
cd moscow
tarantool
```
- `tarantool>`
```lua
box.cfg{replication={'127.0.0.1:3301'}}
```
- Поломаем репликацию конфликтом
- Сначала вставим данные на реплике
```lua
box.space.test:insert{2, 'value2'}
```
- Потом на лидере
```lua
box.space.test:insert{2, 'value2'}
```
- Проверим статус репликации
```
box.info.replication
```
- Ожидаем увидеть сообщение
```
Duplicate key exists in unique index "pkey"
```
- Ребутстрап:
    - Запомним uuid, cluster.uuid в таблице box.info
    ```lua
    box.info.uuid
    box.info.cluster.uuid
    ```
    - Погасим реплику Ctrl-C
    - Удалим файлы *.snap, *.xlog
    - `$`
    ```
    rm -rf *.snap *.xlog
    ```
    - `$`
    ```
    tarantool
    ```
    - `tarantool>`
    ```
    box.cfg{replication={'127.0.0.1:3301'}, instance_uuid='XXX', replicaset_uuid='YYY'}
    ```
