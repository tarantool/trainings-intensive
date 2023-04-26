---
title: Tarantool Topology
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
- Настроить мониторинг
- Запустить http-сервер

## Часть 2 — Масштабирование

### Создание репликасета

- Создадим директорию `cluster`.
- `schema.lua`
```lua
if box.info.ro then
    return
end

box.schema.user.grant('guest', 'super', nil, nil, {if_not_exists=true}) -- grant replication role

box.schema.space.create('redirector', { if_not_exists=true })
box.space.redirector:format({
    { name='source', type='string' }, -- исходная сслылка
    { name='short', type='string' },  -- сокращенная ссылка 
})

box.space.redirector:create_index(
    'primary',
    {
        parts = { 'source' },
        if_not_exists = true,
    }
)
box.space.redirector:create_index(
    'short',
    {
        parts = { 'short' },
        if_not_exists = true,
    }
)
```
- `moscow-storage.lua`
```lua
box.cfg {
    listen = '127.0.0.1:3301',
    replication = {
        '127.0.0.1:3301',
        '127.0.0.1:3302',
    },
    memtx_dir  = "moscow-storage",
    wal_dir    = "moscow-storage",
    replication_connect_quorum = 1,
    replicaset_uuid = 'aaaaaaaa-0000-4000-a000-000000000000',
    instance_uuid = 'aaaaaaaa-0000-4000-a000-000000000011'
}

require('schema')

require('console').start() os.exit()
```
- `moscow-replica.lua`
```lua
box.cfg {
    read_only=true,
    listen = '127.0.0.1:3302',
    replication = {
        '127.0.0.1:3301',
        '127.0.0.1:3302',
    },
    memtx_dir  = "moscow-replica",
    wal_dir    = "moscow-replica",
    replication_connect_quorum = 1,
    replicaset_uuid = 'aaaaaaaa-0000-4000-a000-000000000000',
    instance_uuid = 'aaaaaaaa-0000-4000-a000-000000000012'
}
require('console').start() os.exit()
```
- Запустим инстансы в разных терминалах
- `$`
```
mkdir moscow-storage
tarantool moscow-storage.lua
```
- `$`
```
mkdir moscow-replica
tarantool moscow-replica.lua
```

- Проверим на обоих узлах, что репликация работает.
- `tarantool>`
```lua
box.info.vclock
box.info.replication
```

- Выполним модифицирущую операцию на лидер узле и убедимся, что данные реплицируются
```lua
box.space.redirector:insert({"http://google.com/?q=tarantool", "9da443d847"})
box.space.redirector:insert({"https://www.tarantool.io/en/doc/latest/", "e5b1f259a8"})
box.space.redirector:insert({"https://db-engines.com/en/system/Tarantool", "ad0e253828"})
box.space.redirector:insert({"https://mcs.mail.ru", "47bfb2302e"})
box.space.redirector:insert({"https://github.com/tarantool/tarantool/", "c1ceeaaf95"})
```
- На другом узле проверим наличие данных
```lua
for _, tuple in box.space.redirector.index.short:pairs() do 
  print(tuple)
end
```

## Часть 3 — Шардирование

- Погасим кластер
- Удалим полностью данные на узлах
- `$`
```
cd moscow-storage
rm -rf *.snap *.xlog
cd ..
cd moscow-replica
rm -rf *.snap *.xlog
cd ..
```
- Установим модуль vshard
```
tarantoolctl rocks install vshard
```
- Добавить поле для хранения ключа шардирования
- `schema.lua`
```lua
if box.info.ro then
    return
end

box.schema.user.grant('guest', 'super', nil, nil, {if_not_exists=true}) -- grant replication role

box.schema.space.create('redirector', { if_not_exists=true })
box.space.redirector:format({
    { name='source',    type='string' }, -- исходная сслылка
    { name='short',     type='string' },  -- сокращенная ссылка 
    { name='bucket_id', type='unsigned' },  -- сокращенная ссылка 
})

box.space.redirector:create_index(
    'primary',
    {
        parts = { 'source' },
        if_not_exists = true,
    }
)
box.space.redirector:create_index(
    'short',
    {
        parts = { 'short' },
        if_not_exists = true,
    }
)

box.space.redirector:create_index(
    'bucket_id',
    {
        parts={ "bucket_id" },
        unique=false, if_not_exists=true,
    }
)
```

- Создадим файл с описание топологии кластера
- Топология `topology.lua`
```lua
return {
    bucket_count = 16,
    sharding = {
        ['aaaaaaaa-0000-4000-a000-000000000000'] = {
            replicas = {
                ['aaaaaaaa-0000-4000-a000-000000000011'] = {
                    name = 'moscow-storage',
                    master=true,
                    uri="guest@127.0.0.1:30011"
                },
                ['aaaaaaaa-0000-4000-a000-000000000012'] = {
                    name = 'moscow-replica',
                    master=false,
                    uri="guest@127.0.0.1:30012"
                },
            }
        },
        ['bbbbbbbb-0000-4000-a000-000000000000'] = {
            replicas = {
                ['bbbbbbbb-0000-4000-a000-000000000021'] = {
                    name='spb-storage',
                    master=true,
                    uri="guest@127.0.0.1:30021"
                },
                ['bbbbbbbb-0000-4000-a000-000000000022'] = {
                    name='spb-replica',
                    master=false,
                    uri="guest@127.0.0.1:30022"
                },
            },
        },
    }
}

```
- `router.lua`
```lua
vshard = require('vshard')
topology = require('topology')
vshard.router.cfg(topology)
require('console').start() os.exit(0)
```
- `moscow-storage.lua`
```lua
vshard = require('vshard')

local topology = require('topology')

vshard.storage.cfg({
        bucket_count = topology.bucket_count,
        sharding     = topology.sharding,

        memtx_dir  = "moscow-storage",
        wal_dir    = "moscow-storage",
        replication_connect_quorum = 1,
    },
    'aaaaaaaa-0000-4000-a000-000000000011')

local schema = require('schema')

require('console').start() os.exit(0)
```

- `moscow-replica.lua`
```lua
vshard = require('vshard')

local topology = require('topology')

vshard.storage.cfg({
        bucket_count = topology.bucket_count,
        sharding     = topology.sharding,

        memtx_dir  = "moscow-replica",
        wal_dir    = "moscow-replica",
    },
    'aaaaaaaa-0000-4000-a000-000000000012')

local schema = require('schema')

require('console').start() os.exit(0)
```
- `spb-storage.lua`

```lua
vshard = require('vshard')

local topology = require('topology')

vshard.storage.cfg({
        bucket_count = topology.bucket_count,
        sharding     = topology.sharding,

        memtx_dir  = "spb-storage",
        wal_dir    = "spb-storage",
    },
    'bbbbbbbb-0000-4000-a000-000000000021')

local schema = require('schema')

require('console').start() os.exit(0)
```

- `spb-replica.lua`
```lua
vshard = require('vshard')

local topology = require('topology')

vshard.storage.cfg({
        bucket_count = topology.bucket_count,
        sharding     = topology.sharding,

        memtx_dir  = "spb-replica",
        wal_dir    = "spb-replica",
    },
    'bbbbbbbb-0000-4000-a000-000000000022')

local schema = require('schema')

require('console').start() os.exit(0)
```

- Запустим кластер:

```
mkdir moscow-storage
tarantool moscow-storage.lua
```

```
mkdir moscow-replica
tarantool moscow-replica.lua
```

```
mkdir spb-storage
tarantool spb-storage.lua
```

```
mkdir spb-replica
tarantool spb-replica.lua
```

- Запустим роутер:

```
tarantool router.lua
```

- Забутстрапим шардинг
- На роутере `tarantool>`
```
vshard.router.bootstrap()
```

### Работа с шардированными данными

- Воспользуемся списком урлов, который мы готовили ранее
- Теперь все операции с данными должны проходить через роутер, с помощью функций vshard.router.callrw, vshard.router.callro
```lua
bucket_id = 1
vshard.router.callrw(bucket_id, 'box.space.redirector:insert', {{"http://google.com/?q=tarantool", "9da443d847", bucket_id}})
bucket_id = 4
vshard.router.callrw(bucket_id, 'box.space.redirector:insert', {{"https://www.tarantool.io/en/doc/latest/", "e5b1f259a8", bucket_id}})
bucket_id = 8
vshard.router.callrw(bucket_id, 'box.space.redirector:insert', {{"https://db-engines.com/en/system/Tarantool", "ad0e253828", bucket_id}})
bucket_id = 12
vshard.router.callrw(bucket_id, 'box.space.redirector:insert', {{"https://mcs.mail.ru", "47bfb2302e", bucket_id}})
bucket_id = 16
vshard.router.callrw(bucket_id, 'box.space.redirector:insert', {{"https://github.com/tarantool/tarantool/", "c1ceeaaf95", bucket_id}})
```
- Запрос данных — по условиям задачи по короткому урлу должны получить оригинальный
- Мы шардировали данные вне зависимости от короткого урла, таким образом мы не знаем на каких шардах как распределены короткие урлы
- Значит нам нужно пройтись по всем шардам — это называется map/reduce

### Запрос данных `map/reduce`

- `vshard.router.routeall` возвращает все репликасеты
    - Функции роутера работают только на роутере
- `replica:call*` вызывает функцию на подходящей реплике

- Поищем короткую ссылку 'e5b1f259a8'
```lua
do
    local resultset = {}
    shards, err = vshard.router.routeall()
    if err ~= nil then
        print(err)
        return
    end
    for uid, replica in pairs(shards) do
        local set = replica:callro('box.space.redirector.index.short:select', {{'e5b1f259a8'}})
        for _, redirect in ipairs(set) do
            resultset[redirect[2]] = redirect[1]
        end
    end
    return resultset
end
```

## Troubleshooting

- `can't chdir to <DIRECTORY>: No such file or directory`
    - Создать нужную &lt;DIRECTORY&gt; для хранения файлов
- `ER_ALREADY_RUNNING: Failed to lock WAL directory`
    - Тарантул уже запущен, найти и убить
        - `pkill -9 tarantool`
    - Каждый тарантул должен иметь свою директорию для хранения данных
- `ER_EXACT_MATCH: Invalid key part count in an exact match (expected 1, got 0)`
    - Не задан пользователь в `uri` ноды
- `Incorrect password supplied for user `
    - Не задан пароль в `uri` репликасета
- `Procedure 'vshard.storage.buckets_discovery' is not defined`
    - Сделать `vshard` модуль глобальным
- `Bucket 1 cannot be found. Is rebalancing in progress?`
    - Проверить что все шарды подняты
    - Если подняты cделать `vshard.router.bootstrap()`
- `Please call box.cfg{} first`
    - Попытка использовать базу до конфигурации
    - Возможно применение схемы до конфигурации
- `Error during discovery`
    - Не все узлы подняты
- `Local replica aaaaaaaa-0000-4000-a000-000000000011 wasn't found in config`
    - Проверить что в топологии нод корректная информация и в скрипте запуска используется имеющийся uuid
- `Only one master is allowed per replicaset`
    - Внутри одного репликасета задано два мастера, должен быть только один
- `Duplicate uri sharding:pass@127.0.0.1:30011`
    - У разных нод назначен одинаковый `uri`
- `ER_READONLY: Can't modify data because this instance is in read-only mode.`
    - В топологии не указан `master` для нужной ноды
- `Duplicate uuid`
    - У двух разных нод в топологии задан одинаковый `uuid`
- `can't initialize storage: Incorrect value for option 'replication': failed to connect to one or more replicas`
    - Проверить что все ноды внутри репликасета запущены
- `Some buckets are not active, retry rebalancing later`
    - Проверить что все шарды подняты
    - Если да сделать `vshard.router.bootstrap()`
- `Cluster is already bootstrapped`
    - Попытка вызывать `vshard.router.bootstrap()` на уже забутстрапленом кластере
- `applier.cc:264 E> error applying row: {type: 'INSERT',
   can't read row
   ER_TUPLE_FOUND: Duplicate key exists in unique index 'primary' in space '_cluster'
`
    - Неконсистетно поднят кластер
    - Следить за конфликтами и перезапускать репликацию
    ```lua
    replication = box.cfg.replication
    box.cfg{replication={}}
    box.cfg{replication=replication, replication_skip_conflict=true}
    ```
