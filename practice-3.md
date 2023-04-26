---
title: Tarantool Maintenance
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

## Часть 3 — мониторинг

- Установим модуль metrics (0.17.0 или новее)
```
tarantoolctl rocks install metrics
```
- Добавим мониторинг сети между узлами приложения
- `router.lua`
```lua
vshard = require('vshard')
local topology = require('topology')
vshard.router.cfg(topology)

local log = require('log')
local fiber = require('fiber')
local yaml = require('yaml')

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network'},
    labels = { alias = 'router' },
}

-- Периодически печатает метрики в консоль
fiber.create(function()
    while true do
        log.info(yaml.encode(metrics.collect{invoke_callbacks = true}))
        fiber.sleep(10)
    end
end)

require('console').start() os.exit(0)
```

- После запуска мы будет периодически видеть сообщения вида
```yaml
- label_pairs:
    alias: router
  timestamp: 1682495738720895
  value: 2
  metric_name: tnt_net_requests_current
- label_pairs:
    alias: router
  timestamp: 1682495738720895
  value: 136232
  metric_name: tnt_net_sent_total
```

- Установим модуль http для возможности работать в режиме HTTP-сервера
```
tarantoolctl rocks install http
```

- Настроим API для работы с Prometheus
- `router.lua`
```lua
vshard = require('vshard')
local topology = require('topology')
vshard.router.cfg(topology)

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network'},
    labels = { alias = 'router' },
}

local httpd = require('http.server').new('0.0.0.0', 8080)
local prometheus = require('metrics.plugins.prometheus')
httpd:route({ path = '/metrics' }, prometheus.collect_http)
httpd:start()

require('console').start() os.exit(0)
```

- Проверим ответ
```bash
curl -X GET localhost:8080/metrics
```
```
# HELP tnt_net_requests_current Pending requests
# TYPE tnt_net_requests_current gauge
tnt_net_requests_current{alias="router"} 2
# HELP tnt_net_sent_total Totally sent in bytes
# TYPE tnt_net_sent_total counter
tnt_net_sent_total{alias="router"} 136232
```

- Добавим метрики на хранилища
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

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network', 'operations', 'replicas', 'slab'},
    labels = { alias = 'moscow-storage' },
}

local httpd = require('http.server').new('0.0.0.0', 8081)
local prometheus = require('metrics.plugins.prometheus')
httpd:route({ path = '/metrics' }, prometheus.collect_http)
httpd:start()

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

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network', 'operations', 'replicas', 'slab'},
    labels = { alias = 'moscow-replica' },
}

local httpd = require('http.server').new('0.0.0.0', 8082)
local prometheus = require('metrics.plugins.prometheus')
httpd:route({ path = '/metrics' }, prometheus.collect_http)
httpd:start()

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

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network', 'operations', 'replicas', 'slab'},
    labels = { alias = 'spb-storage' },
}

local httpd = require('http.server').new('0.0.0.0', 8083)
local prometheus = require('metrics.plugins.prometheus')
httpd:route({ path = '/metrics' }, prometheus.collect_http)
httpd:start()

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

local metrics = require('metrics')
metrics.cfg{
    include = {'info', 'network', 'operations', 'replicas', 'slab'},
    labels = { alias = 'spb-replica' },
}

local httpd = require('http.server').new('0.0.0.0', 8084)
local prometheus = require('metrics.plugins.prometheus')
httpd:route({ path = '/metrics' }, prometheus.collect_http)
httpd:start()

require('console').start() os.exit(0)
```

- Зададим конфигурацию Prometheus
```
prometheus.yml
```
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "tarantool"
    static_configs:
      - targets:
        - "localhost:8080"
        - "localhost:8081"
        - "localhost:8082"
        - "localhost:8083"
        - "localhost:8084"
    metrics_path: "/metrics"
```

- Запустим Prometheus
```
docker run -d --name prometheus --network host -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus:v2.17.2
```

- Зададим конфигурацию Grafana

```
grafana.yml
```
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: localhost:9090
```

- Запустим Grafana
```
docker run -d --name grafana --network host -v `pwd`/grafana.yml:/etc/grafana/provisioning/datasources/automatic.yaml -e GF_SECURITY_DISABLE_INITIAL_ADMIN_CREATION=true -e GF_AUTH_ANONYMOUS_ENABLED=true -e GF_AUTH_ANONYMOUS_ORG_ROLE='Admin' -e GF_AUTH_DISABLE_SIGNOUT_MENU=true -e GF_AUTH_DISABLE_LOGIN_FORM=true grafana/grafana:8.1.3
```

- Откроем в браузере адрес `localhost:3000`
- Импортируем стандартный дашборд с идентификатором `13054` ([подробнее](https://www.tarantool.io/en/doc/latest/book/monitoring/grafana_dashboard/)).
