---
title: Tarantool Arch
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
- Запустить http-сервер

## Часть 1 — Хранилище

### План

- Конфигурация базы данных
- Схема хранения
- Запросы

### Конфигурация базы данных

- `init.lua`
```lua
box.cfg{}
```
- Инициализация файлов

### Создание спейса для хранения

Для решение задачи сделаем спейс со следующими полями:
- оригинальная ссылка
- сокращённая ссылка

- создание спейса
```lua
box.schema.space.create('redirector', { if_not_exists=true })
```

Флаг if_not_exists подавляет ошибку в случае существования спейса.

Укажем поля и типы полей для спейса:
```lua
box.space.redirector:format({
    { name='source', type='string' }, -- исходная сслылка
    { name='short', type='string' },  -- сокращенная ссылка 
})
```

### Создание индексов 

Для просто хранения данных нам нужен первичный индекс.

Сделаем первичный ключ по полю с оригинальной ссылкой.

```lua
box.space.redirector:create_index(
    'primary',
    {
        parts = { 'source' },
        if_not_exists = true,
    }
)
```

В случае, когда пользователь переходит по сокращенной ссылке, нам надо найти оригинальную.

```lua
box.space.redirector:create_index(
    'short',
    {
        parts = { 'short' },
        if_not_exists = true,
    }
)
```

### Создание сокращенной ссылки

Для вставки новой ссылки сделаем сокращённый хеш ссылки:

```bash
$ echo -n "https://go.mail.ru/search\?q\=tarantool" | shasum | head -c 10
```

Cоздадим запрос на вставку

```lua
box.space.redirector:insert({"https://go.mail.ru/search?q=tarantool", "0f74d9fd75"})
```

Если теперь сделать повторную вставку даже с другим url — получим ошибку по индексу по полю с сокращенной ссылкой.

```lua
box.space.redirector:insert({"https://habr.com", "0f74d9fd75"})
```

```
Duplicate key exists in unique index 
```

Вставим некоторое множество ссылок:

```lua
box.space.redirector:insert({"http://google.com/?q=tarantool", "9da443d847"})
box.space.redirector:insert({"https://www.tarantool.io/en/doc/latest/", "e5b1f259a8"})
box.space.redirector:insert({"https://db-engines.com/en/system/Tarantool", "ad0e253828"})
box.space.redirector:insert({"https://mcs.mail.ru", "47bfb2302e"})
box.space.redirector:insert({"https://github.com/tarantool/tarantool/", "c1ceeaaf95"})
```

### Запрос ссылки по сокращенной части

Для поиска оригинальной ссылки по сокращенной воспользуемся итератором по индексу:

```lua
box.space.redirector.index.short:get("0f74d9fd75")
```

### Для перечисления всех ссылок, что есть в нашей базе

Такой запрос можно сделать несколькими способами:
```lua
for _, tuple in box.space.redirector.index.short:pairs() do 
  print(tuple)
end
```
