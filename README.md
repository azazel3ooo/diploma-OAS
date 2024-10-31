# REST-like API

Данный стандарт является компиляцией существующих стандартов json-ld, HATEOS, Hydra,
а так же добавлением собственных правил, исходя из опыта и задач систем.

{вводная зачем, кто, как}
* Документация ведется по спецификации [OpenAPI 3.0.3](https://swagger.io/specification/)

## Требования и рекомендации
* API оперирует только обменными моделями. API никак не связано со структурой БД, вообще, совсем, никогда.
* Мы используем http2 - это решает основную часть проблем из-за которых появился json:api, именно поэтому мы не используем эту спецификацию как есть
* Рекомендуется использовать тонкие объекты
* Не встраивать в объект коллекции истинных объектов, особенно склонных к большому размеру. Т.е., например, допустимо включение enum. Вместо этого работу с коллекциями осуществлять с помощью дополнительных адресов. 
* В OAS уже описаны форматы [uuid, email, дата, ip](https://swagger.io/docs/specification/data-models/data-types/#string)
* В качестве идентификатора используется [IRI reference](https://www.w3.org/TR/json-ld11/#terms-imported-from-other-specifications)
* Структура формируется в виде совместимом со спецификацией [json-ld 1.1](https://www.w3.org/TR/json-ld11/)
* По умолчанию mime-тип `application/json`
* Описание ведётся в раздельном виде, корневым файлом является `openapi.yaml`; файл `bundle.yaml` - собранный файл и нужен для совместимости
* Не забываем удалять файлы примеров `example.yaml`, `example_bundle.yaml`
#### Правила именования
- Элементы в url path - `kebab-case`. Пример `{basePath}/big-cities/new-york`
- Элементы в url query - `camelCase`. Пример `{basePath}/big-cities?settledAt=1970-01-01`
- Переменные в url (template) - `camelCase`. Пример `{basePath}`
- Наименование объектов - `PascalCase`
- Свойства объектов в теле json - `snake_case`
#### Дата по умолчанию в формате ISO 8601
- UTC зона по умолчанию
- `YYYY-MM`  `2020-06`
- `YYYY-MM-DD`  `2020-06-02`
- `YYYY-MM-DDThh:mm:ss±hh`  `2020-06-02T15:22:00+00:00`
#### Ответ: Статус коды
- Для всех эндпоинтов необходимо возвращать статус-коды.
- Использовать стандартные коды ответов ([Коды ответа HTTP](https://developer.mozilla.org/ru/docs/Web/HTTP/Status))
- Каждый сервис потенциально должен уметь возвращать `500`.
- Каждый сервис, который требует минимум один обязательный параметр на вход, должен уметь возвращать `4xx` с описанием.
- Каждый секьюрный сервис должен уметь возвращать `401`.

##### GET
- Если данных нет, каждый сервис должен возвращать `404`.
- Если данных найдено в результате фильтрации, каждый сервис должен возвращать `204`.
- Каждая коллекция должна вернуть `200`, если возвращается полностью.
- Каждая коллекция должна вернуть `206`, если возвращается частично.

<details>
<summary>Пример простого ответа</summary>

```json
{
    "@id": "http://example.com/events/ecom-party-2023",
    "title": "ECom party"
}
```
</details>
<details>
<summary>Пример плоской коллекции объектов</summary>

```json
[
    { "@id": "http://example.com/events/ecom-party-2023", ... },
    ...
]
```
Может сопровождаться комплектом заголовков пагинации

```http
X-Total-Count: 542
X-Page: 3
X-Per-Page: 25
X-Total-Pages: 22
```
или HATEOS-версия

```http
X-Total-Count: 542
Link: <http://localhost/api/books?offset=15&limit=5>; rel="next",
      <http://localhost/api/books?offset=50&limit=3>; rel="last",
      <http://localhost/api/books?offset=0&limit=5>; rel="first",
      <http://localhost/api/books?offset=5&limit=5>; rel="prev"
```
</details>
<details>
<summary>Пример коллекции в обёртке hydra:Collection</summary>

```json
{
    "@type": "hydra:Collection",
    "@context": "http://www.w3.org/ns/hydra/context.jsonld",
    "totalItems": 4975,
    "member": [
        { "@id": "http://example.com/event/ecom-party-2023" },
        ...
    ]
}
```
</details>
<details>
<summary>Пример коллекции в обёртке hydra:PartialCollectionView</summary>

```json
{
    "@id": "http://api.example.com/an-issue/comments", // iri коллекции, основной урл
    "@context": "http://www.w3.org/ns/hydra/context.jsonld",
    "@type": "hydra:Collection",
    "member": [ 
        {
            "@id": "http://example.com/an-issue/comments/max21" // iri объекта коллекции
        },
        ...
    ],
    "totalItems": 4975,
    "hydra:view": {
        "@id": "/an-issue/comments?:page=3", // iri этого запроса, может быть относительным за счет наличия основного iri
        "@type": "hydra:PartialCollectionView",
        "first": "/an-issue/comments?:page=1",
        "previous": "/an-issue/comments?:page=2",
        "next": "/an-issue/comments?:page=4",
        "last": "/an-issue/comments?:page=498"
    }
}
```
</details>

##### POST
-   Возвращаем `201` после создания ресурса.
-   Возвращаем `202`, если запрос обрабатывается асинхронно.
##### PATCH & PUT
-   Возвращаем `200` после обновления ресурса, если ресурс отличается в результате обработки на сервере.
-   Возвращаем `204` после обновления ресурса, если ресурс не отличается в результате обработки на сервере.
-   Возвращаем `202`, если запрос обрабатывается асинхронно.
#### Ответ: Коды  ошибок
Вместе с HTTP кодами ответов в теле необходимо указать код ошибки. Это позволит потребителю понять, что именно пошло не так, и правильно среагировать. Ответ с ошибкой может содержать бизнес код ошибки с описанием причины падения. Все эти коды должны быть описаны в документации API.

В соответствии с [RFC7807](https://www.rfc-editor.org/rfc/rfc7807).
<details>
<summary>Общий пример</summary>

```json
{
    "type": "urn:problem-type:rk:outOfCredit",
    "title": "You do not have enough credit.",
    "detail": "Your current balance is 30, but that costs 50.",
    "instance": "/account/12345/msgs/abc",
    "status": 400
    ...
}
```
</details>
<details>

<summary>Пример нарушения валидации. Код `422`</summary>

```json
{
    "type": "urn:problem-type:rk:badRequest",
    "title": "Unprocessable Entity",    
    "detail": "boolean: This value should be of type bool.\npropertyPath1: This value should be of type string.",
    "issues": [
        {
            "type": "urn:problem-type:rk:input-validation:schemaViolation",
            "in": "path",
            "name": "boolean_property_path_1",
            "title": "This value should be of type bool.",
        },
        {
            "type": "urn:problem-type:rk:input-validation:schemaViolation",
            "in": "body",
            "name": "author.property_path_2",
            "title": "This value should be of type string."
        }
    ]
}
```
</details>

#### Фильтрация  и сортировка
propertyPath - пусть к свойству, разделенной символом точки. Пример `author.rating`
##### Boolean
Синтаксис `?propertyPath=<true|false|1|0>`
##### Numeric
Синтаксис `?propertyPath=<int|bigint|decimal...>`
##### Range
Синтаксис `?propertyPath=value` или `?propertyPath[<lt|gt|lte|gte>]=value`

Пример `?propertyPath[gte]=5.5`
##### Date
Синтаксис аналогично [Range](#range)
##### In, Not in
Синтаксис `?propertyPath[in]=<array via |>`, `?propertyPath[not_in]=<array via |>`
##### Exists
Синтаксис `?:exists[propertyPath]=<true|false|1|0>`
##### Order
Синтаксис `?:order[propertyPath]=<asc|desc>`
##### Pagination
Синтаксис `?:offset=<int>&:limit=<int>`


## Тулинг

#### Redocly CLI
Набор утилит для работы с openapi файлами

[Где брать](https://redocly.com/docs/cli/)

##### Примеры:
Собрать всё в бандл. Заменит файл openapi.yaml
```sh
openapi bundle -o bundle openapi.yaml
```
Разобрать бандл по файлам. Заменит файл openapi.yaml, остальные сущесвтующие пропустит, сообщив какие именно!
```sh
openapi split bundle.yaml --separator="//" --outDir ./
```
Запусть preview-server
```sh
openapi preview-docs openapi.yaml
```
Запусть проверку файла (линтер)
```sh
openapi lint openapi.yaml
```


#### Stoplight Studio
UI для работы с openapi. Умеет выести schemes в отдельных файлах. Из минусов - не умеет в [Links](https://swagger.io/docs/specification/links/)

[Где брать](https://stoplight.io/studio)
