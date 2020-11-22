# API - медиатор для микросервисов

Данный документ содержит описание работы и информацию о развертке микросервиса API.

Название сервиса: `gateway`

Общедоступный IP-адрес: `20.52.2.24`

Структура сервиса:

| Файл                 | Описание                                                |
| -------------------- | ------------------------------------------------------- |
| `gateway.py`         | Код микросервиса                                        |
| `config.yml`         | Конфигурационный файл со строкой подключения к RabbitMQ |
| `run.sh`             | Файл для запуска сервиса из Docker контейнера           |
| `requirements.txt`   | Верхнеуровневые зависимости                             |
| `Dockerfile`         | Описание сборки контейнера сервиса                      |
| `docker-compose.yml` | Изолированная развертка сервиса вместе с RabbitMQ       |
| `README.md`          | Описание микросервиса                                   |

## API

### HTTP

Регистрация пользователей:

```rst
POST http://localhost:8000/register HTTP/1.1
Content-Type: application/json

{
    "login": <login>,
    "password": <password>
}

Response (Сообщение об успехе, либо ошибка):
Status: 201 - успех, 400 - ошибка регистрации
Content-Type: application/json

{
    "message": <msg>
}
```

Вход:

```rst
POST http://localhost:8000/login HTTP/1.1
Content-Type: application/json

{
    "login": <login>,
    "password": <password>
}

Response (Correct credentials):
Status: 202 - успех
Content-Type: application/json

{
    "token": <token>
}

Response (Wrong credentials):
Status: 400 - неверный логин или пароль
Content-Type: application/json

{
    "message": <msg>
}
```

**В теле всех остальных запросов должен присутствовать JWT токен, если это зарегистрированный пользователь.**

Получение ленты событий:

```rst
POST http://localhost:8000/feed HTTP/1.1
Content-Type: application/json

{
    "token": <token>, (Может быть не указан, если пользователь не авторизован)
    "tags": [..] - Теги для фильтрации (могут быть не указаны)
}

Response:
Status: 200 - успех
Content-Type: application/json

[
    {
        "_id": <event_id>,
        "title": <title>,
        "location": <location>,
        "startDate": <startDate>,
        "endDate": <endDate>,
        "description": <description>,
        "meta": <meta>,
        "tags": [..] - list
    },
    {
        "_id": <event_id>,
        "title": <title>,
        "location": <location>,
        "startDate": <startDate>,
        "endDate": <endDate>,
        "description": <description>,
        "meta": <meta>,
        "tags": [..] - list
    }
    ...
]

Response (При неверном токене):
Status: 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Получение определенного события:

```rst
GET http://localhost:8000/feed/<string:event_id>?token=<token> HTTP/1.1

Query parameters:
{
    "token": <token> (Может быть не указан, если пользователь не авторизован)
}

Response (событие):
Status: 200
Content-Type: application/json

{
    "_id": <event_id>,
    "title": <title>,
    "location": <location>,
    "startDate": <startDate>,
    "endDate": <endDate>,
    "description": <description>,
    "meta": <meta>,
    "tags": [..] - list
}

Response (ошибка):
Status: 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Получение интересов пользователя:

```rst
GET http://localhost:8000/profile/interests?token=<token> HTTP/1.1

Query parameters:
{
    "token": <token>
}

Response:
Status: 200 - успех
Content-Type: application/json

{
    "interests": ['tag1', 'tag2', ...],
    "ind": [True, False, False, ...]
}

Response (Не передан токен, либо он неверный):
Status: 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Добавление новой анкеты интересов:

```rst
POST http://localhost:8000/profile/interests HTTP/1.1
Content-Type: application/json

{
    "token": <token>,
    "interests": ['tag1', 'tag2', ...],
    "ind": [True, False, False, ...]
}

Response (Сообщение об успешном добавлении, либо ошибка):
Status: 200 - успех, 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Изменение интересов:

```rst
PUT http://localhost:8000/profile/interests HTTP/1.1
Content-Type: application/json

{
    "token": <token>,
    "interests": ['tag1', 'tag2', ...],
    "ind": [True, False, False, ...]
}

Response (Сообщение об успешном изменении, либо ошибка):
Status: 200 - успех, 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

**Далее описаны запросы для лайков, для добавление в избранное заменить like на favorite в адресе.**

Реакционное событие (добавление лайка):

```rst
POST http://localhost:8000/reaction/like HTTP/1.1
Content-Type: application/json

{
    "token": <token>,
    "event_id": <event_id>
}

Response (Сообщение об успехе, либо ошибка):
Status: 200 - успех, 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Реакционное событие (получение всех лайков/наличие лайка у мероприятия):

```rst
GET http://localhost:8000/reaction/like?token=<token>&event_id=<event_id> HTTP/1.1
Content-Type: application/json

Query parameters:
{
    "token": <token>,
    "event_id": <event_id> - Указывается, если нужно узнать наличие лайка у конкретного мероприятия
}

Response (успех, запрос на все лайки):
Status: 200
Content-Type: application/json

[
    event_id1, - id, лайкнутых всех событий
    event_id2,
    ...
]

Response (успех, конкретное мерориятие):
Status: 200
Content-Type: application/json

{
    "value": (true or false)
}

Response (ошибка):
Status: 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

Реакционное событие (удаление лайка):

```rst
DELETE http://localhost:8000/reaction/like HTTP/1.1
Content-Type: application/json

{
    "token": <token>,
    "event_id": <event_id>
}

Response (Сообщение об успехе, либо ошибка):
Status: 200 - успех, 401 - пользователь не авторизован (токен не передан), 403 - неверный токен
Content-Type: application/json

{
    "message": <msg>
}
```

## Развертывание и запуск

### Локальный запуск

Для локального запуска микросервиса требуется запустить контейнер с RabbitMQ.

```bat
docker-compose up -d
```

Затем из папки микросервиса вызвать

```bat
nameko run gateway
```

### Запуск в контейнере

Чтобы запустить микросервис в контейнере вызовите команду:

```bat
docker-compose up
```

> если вы не хотите просмотривать логи, добавьте флаг `-d` в конце
