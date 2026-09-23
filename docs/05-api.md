# API Delivery Orchestration Platform

Документ содержит REST API-контракты платформы оркестрации доставки.

Контракты будут уточняться по мере проектирования системы.

---

## 1. Расчёт вариантов доставки

### POST /delivery-offers

Рассчитывает доступные варианты доставки для конкретного `Shipment`.

API используется системой Checkout во время оформления заказа.

Платформа получает параметры отправления и адрес назначения, определяет подходящих перевозчиков, получает от них тарифы и сроки и возвращает нормализованный список `DeliveryOffer`.

### Request

```json
{
  "shipmentId": "shp-501",
  "originLocationId": "warehouse-12",
  "destination": {
    "city": "Москва",
    "street": "Ленинский проспект",
    "house": "10"
  },
  "weight": {
    "value": 5.2,
    "unit": "KG"
  },
  "dimensions": {
    "length": 40,
    "width": 30,
    "height": 20,
    "unit": "CM"
  },
  "contentType": "ELECTRONICS",
  "declaredValue": {
    "amount": 85000,
    "currency": "RUB"
  }
}
```

### Поля запроса

| Поле | Описание |
|---|---|
| `shipmentId` | Идентификатор отправления |
| `originLocationId` | Идентификатор точки отправления / склада |
| `destination` | Адрес назначения |
| `weight` | Вес отправления и единица измерения |
| `dimensions` | Габариты отправления |
| `contentType` | Тип содержимого |
| `declaredValue` | Объявленная ценность отправления |

### Поиск пунктов выдачи

Для поиска доступных ПВЗ платформа использует адрес назначения из поля `destination`.

Адрес преобразуется в географические координаты. Поиск выполняется в пределах радиуса, заданного конфигурацией платформы.

Радиус поиска не передаётся в текущей версии API.

В ответ включаются подходящие предложения `DeliveryOffer` с информацией о ПВЗ, доступных для данного Shipment.
---

### Идентификация отправления

Для расчёта доставки используется `shipmentId`.

Передача `orderId` не требуется, поскольку расчёт может выполняться до создания заказа.

Связь Shipment с Order устанавливается позже, после получения события `OrderReadyForDelivery`.

## 2. Успешный ответ

### 200 OK

```json
{
  "shipmentId": "shp-501",
  "offers": [
    {
      "offerId": "off-1001",
      "deliveryType": "COURIER",
      "price": {
        "amount": 450,
        "currency": "RUB"
      },
      "estimatedDeliveryFrom": "2026-10-02",
      "estimatedDeliveryTo": "2026-10-03",
      "validUntil": "2026-10-01T15:00:00+03:00"
    },
    {
      "offerId": "off-1002",
      "deliveryType": "PICKUP_POINT",
      "pickupPoint": {
        "pickupPointId": "pvz-123",
        "name": "ПВЗ",
        "address": "Москва, ул. Профсоюзная, д. 10",
        "latitude": 55.6781,
        "longitude": 37.5623,
        "workingHours": "09:00-21:00"
      },
      "price": {
        "amount": 300,
        "currency": "RUB"
      },
      "estimatedDeliveryFrom": "2026-10-03",
      "estimatedDeliveryTo": "2026-10-04",
      "validUntil": "2026-10-01T15:00:00+03:00"
    }
  ]
}
```

### Поля DeliveryOffer
### Поля ответа
| Поле | Описание |
|---|---|
| `shipmentId` | Идентификатор отправления, для которого выполнен расчёт доставки |
| `offers` | Массив доступных вариантов доставки. Может быть пустым, если подходящие варианты отсутствуют |
| `offers[].offerId` | Уникальный идентификатор рассчитанного предложения доставки |
| `offers[].deliveryType` | Способ доставки: `COURIER` или `PICKUP_POINT` |
| `offers[].pickupPoint` | Объект с информацией о пункте выдачи. Возвращается только для `PICKUP_POINT` |
| `offers[].pickupPoint.pickupPointId` | Внутренний идентификатор пункта выдачи |
| `offers[].pickupPoint.name` | Наименование пункта выдачи |
| `offers[].pickupPoint.address` | Адрес пункта выдачи |
| `offers[].pickupPoint.latitude` | Географическая широта пункта выдачи |
| `offers[].pickupPoint.longitude` | Географическая долгота пункта выдачи |
| `offers[].pickupPoint.workingHours` | Режим работы пункта выдачи |
| `offers[].price` | Объект со стоимостью доставки |
| `offers[].price.amount` | Рассчитанная стоимость доставки |
| `offers[].price.currency` | Код валюты в формате ISO 4217, например `RUB` |
| `offers[].estimatedDeliveryFrom` | Минимальная ожидаемая дата доставки в формате `YYYY-MM-DD` |
| `offers[].estimatedDeliveryTo` | Максимальная ожидаемая дата доставки в формате `YYYY-MM-DD` |
| `offers[].validUntil` | Дата и время окончания действия предложения в формате ISO 8601 |

### Срок действия DeliveryOffer

`validUntil` определяет срок, до которого предложение может быть выбрано и подтверждено пользователем.

Перед оплатой Checkout выполняет повторную проверку предложения через `POST /delivery-offers/{offerId}/validate`.

Если предложение было подтверждено до истечения `validUntil`, последующее истечение срока действия не отменяет согласованные условия доставки.

`DeliveryOffer` сохраняется и используется при последующем создании `Delivery`.

## 3. Отсутствие доступных вариантов

Если запрос корректен и расчёт успешно выполнен, но ни один перевозчик не может доставить Shipment, платформа возвращает:

### 200 OK

```json
{
  "shipmentId": "shp-501",
  "offers": []
}
```

Пустой массив означает, что расчёт был выполнен успешно, но подходящих вариантов доставки нет.

---

## 4. HTTP-коды ответа

| HTTP-код | Ситуация |
|---|---|
| `200 OK` | Расчёт успешно выполнен |
| `200 OK` + `offers: []` | Расчёт выполнен, но доступных вариантов нет |
| `400 Bad Request` | Request содержит некорректные или отсутствующие обязательные данные |
| `500 Internal Server Error` | Непредвиденная внутренняя ошибка платформы |
| `503 Service Unavailable` | Расчёт временно невозможно выполнить из-за недоступности критичных внешних зависимостей |

### Разница между отсутствием вариантов и технической недоступностью

Если перевозчики успешно обработали запрос, но ни один из них не подходит:

```text
200 OK
offers = []
```

Если платформа не смогла выполнить расчёт, например все необходимые API перевозчиков недоступны:

```text
503 Service Unavailable
```

Пустой список вариантов не должен использоваться для маскировки технической ошибки.

---
## 5. Формат ошибок валидации

При ошибках валидации платформа возвращает `400 Bad Request`.

Ответ должен содержать общий код ошибки и список невалидных полей.

Пример:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "errors": [
    {
      "field": "originLocationId",
      "message": "Field is required"
    },
    {
      "field": "destination",
      "message": "Field is required"
    },
    {
      "field": "weight.value",
      "message": "Value must be greater than 0"
    }
  ]
}
```
## 6. Повторная проверка DeliveryOffer

### POST /delivery-offers/{offerId}/validate

Проверяет актуальность ранее рассчитанного DeliveryOffer перед оплатой заказа.

Проверка может завершиться одним из трёх бизнес-результатов:

- `VALID` — условия не изменились;
- `CHANGED` — стоимость, срок или другие значимые условия изменились;
- `UNAVAILABLE` — выбранный вариант больше недоступен.
### Response

Во всех случаях API возвращает единый формат ответа.

| Поле | Описание |
|---|---|
| `status` | Результат проверки: `VALID`, `CHANGED` или `UNAVAILABLE` |
| `previousOfferId` | Идентификатор исходного предложения |
| `currentOffer` | Актуальное предложение. Может быть `null`, если вариант больше недоступен |

---

### Offer актуален

`200 OK`

```json
{
  "status": "VALID",
  "previousOfferId": "off-1001",
  "currentOffer": {
    "offerId": "off-1001",
    "deliveryType": "COURIER",
    "price": {
      "amount": 450,
      "currency": "RUB"
    },
    "estimatedDeliveryFrom": "2026-09-20",
    "estimatedDeliveryTo": "2026-09-21",
    "validUntil": "2026-09-18T14:00:00+04:00"
  }
}
```

---

### Условия изменились

`200 OK`

```json
{
  "status": "CHANGED",
  "previousOfferId": "off-1001",
  "currentOffer": {
    "offerId": "off-1002",
    "deliveryType": "COURIER",
    "price": {
      "amount": 500,
      "currency": "RUB"
    },
    "estimatedDeliveryFrom": "2026-09-21",
    "estimatedDeliveryTo": "2026-09-22",
    "validUntil": "2026-09-18T14:00:00+04:00"
  }
}
```

Пользователь должен повторно подтвердить изменившиеся условия перед оплатой.

---

### Offer больше недоступен

`200 OK`

```json
{
  "status": "UNAVAILABLE",
  "previousOfferId": "off-1001",
  "currentOffer": null
}
```

В этом случае Checkout должен выполнить новый расчёт:

```text
POST /delivery-offers
```

---

### Техническая невозможность проверки

Если проверить актуальность предложения невозможно из-за недоступности критичных внешних зависимостей:

`503 Service Unavailable`

`UNAVAILABLE` означает, что проверка успешно выполнена и вариант действительно больше недоступен.

`503 Service Unavailable` означает, что платформа не смогла выполнить саму проверку.

### Подтверждение предложения пользователем

Успешная проверка DeliveryOffer не означает, что пользователь подтвердил его выбор.

После получения результата `VALID` Checkout позволяет пользователю продолжить оформление заказа.

Если результат проверки — `CHANGED`, пользователь должен подтвердить обновлённые условия и новый `offerId`.

Факт подтверждения, идентификатор выбранного предложения и время подтверждения сохраняются на стороне Order Service.

Delivery Orchestration Platform сохраняет рассчитанный DeliveryOffer и предоставляет его данные для последующего создания доставки.

Если срок действия предложения истёк до подтверждения пользователем, необходимо выполнить повторную проверку.

## 7. Получение состояния доставки

### GET /deliveries/{deliveryId}

Возвращает текущее состояние ранее созданной доставки.

Endpoint может использоваться внутренними системами маркетплейса для получения актуального состояния Delivery.

### Path parameters

| Параметр | Описание |
|---|---|
| `deliveryId` | Идентификатор доставки |

### Успешный ответ

`200 OK`

```json
{
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "deliveryOfferId": "off-1001",
  "deliveryType": "COURIER",
  "status": "IN_TRANSIT",
  "price": {
    "amount": 450,
    "currency": "RUB"
  },
  "estimatedDeliveryFrom": "2026-09-20",
  "estimatedDeliveryTo": "2026-09-21",
  "createdAt": "2026-09-17T15:30:00+04:00",
  "updatedAt": "2026-09-18T10:15:00+04:00"
}
```

Для доставки в ПВЗ дополнительно возвращается `pickupPointId`:

```json
{
  "deliveryId": "dlv-1002",
  "orderId": "ord-502",
  "shipmentId": "shp-502",
  "deliveryOfferId": "off-1002",
  "deliveryType": "PICKUP_POINT",
  "pickupPointId": "pvz-123",
  "status": "READY_FOR_PICKUP",
  "price": {
    "amount": 300,
    "currency": "RUB"
  },
  "estimatedDeliveryFrom": "2026-09-20",
  "estimatedDeliveryTo": "2026-09-21",
  "createdAt": "2026-09-17T16:00:00+04:00",
  "updatedAt": "2026-09-20T12:40:00+04:00"
}
```

### Поля ответа

| Поле | Описание |
|---|---|
| `deliveryId` | Идентификатор доставки |
| `orderId` | Идентификатор заказа |
| `shipmentId` | Идентификатор отправления |
| `deliveryOfferId` | Идентификатор выбранного DeliveryOffer |
| `deliveryType` | Тип доставки: `COURIER` или `PICKUP_POINT` |
| `pickupPointId` | Идентификатор ПВЗ. Заполняется для доставки в пункт выдачи |
| `status` | Текущий нормализованный статус доставки |
| `price` | Стоимость доставки |
| `estimatedDeliveryFrom` | Минимальная ожидаемая дата доставки |
| `estimatedDeliveryTo` | Максимальная ожидаемая дата доставки |
| `createdAt` | Дата и время создания Delivery |
| `updatedAt` | Дата и время последнего изменения Delivery |

### Возможные статусы Delivery

```text
CREATED
ACCEPTED
PICKED_UP
IN_TRANSIT
OUT_FOR_DELIVERY
READY_FOR_PICKUP
DELIVERY_FAILED
RETURN_IN_PROGRESS
DELIVERED
CANCELLED
RETURNED
```

Статусы перевозчиков нормализуются платформой и преобразуются во внутреннюю модель состояний Delivery.

### Delivery не найдена

Если `deliveryId` не существует:

`404 Not Found`

```json
{
  "code": "DELIVERY_NOT_FOUND",
  "message": "Delivery not found"
}
```

### Внутренняя ошибка

При непредвиденной ошибке платформы:

`500 Internal Server Error`

## 8. Отмена доставки

### POST /deliveries/{deliveryId}/cancel

Отменяет Delivery, если её текущее состояние допускает обычную отмену.

Delivery не удаляется из системы. В результате операции её статус изменяется на `CANCELLED`.

### Path parameters

| Параметр | Описание |
|---|---|
| `deliveryId` | Идентификатор доставки |

### Допустимые состояния для отмены

Обычная отмена разрешена только в состояниях:

```text
CREATED
ACCEPTED
```
### Особенности отмены в зависимости от статуса

Если `Delivery` находится в статусе `CREATED` и доставка ещё не была подтверждена Carrier, платформа может отменить Delivery внутри своей системы.

Если `Delivery` находится в статусе `ACCEPTED`, обычная отмена требует взаимодействия с Carrier.

Последовательность:

```text
ACCEPTED
   ↓
запрос отмены Carrier
   ↓
Carrier подтвердил отмену
   ↓
CANCELLED
```
Допустимые переходы:

```text
CREATED  → CANCELLED
ACCEPTED → CANCELLED
```

После передачи отправления перевозчику обычная отмена больше не выполняется.

Начиная со статуса:

```text
PICKED_UP
```

необходимо запускать процесс возврата:

```text
PICKED_UP
    ↓
RETURN_IN_PROGRESS
    ↓
RETURNED
```

---

### Успешная отмена

`200 OK`

```json
{
  "deliveryId": "dlv-1001",
  "status": "CANCELLED",
  "updatedAt": "2026-09-17T20:30:00+04:00"
}
```

### Отмена во время создания доставки у Carrier

Если Delivery находится в статусе `CREATED`, но задание оформления уже выполняется, платформа сохраняет запрос на отмену.

В этом случае возвращается:

`202 Accepted`

```json
{
  "deliveryId": "dlv-1001",
  "status": "CREATED",
  "cancellationPending": true
}
```

Ответ означает, что запрос принят, но фактическая отмена ещё не подтверждена.

Повторный запрос отмены при `cancellationPending = true` не запускает новую операцию и возвращает тот же результат.

Текущее состояние доставки можно получить через `GET /deliveries/{deliveryId}`.
---

### Delivery уже отменена

Повторный запрос отмены для Delivery со статусом `CANCELLED` не должен создавать дополнительные действия.

Платформа возвращает текущее состояние:

`200 OK`

```json
{
  "deliveryId": "dlv-1001",
  "status": "CANCELLED",
  "updatedAt": "2026-09-17T20:30:00+04:00"
}
```

Таким образом повторный запрос отмены не изменяет результат операции.

### Неопределённый результат отмены

Если при обращении к Carrier произошёл timeout и платформа не может установить результат отмены, она возвращает:

`503 Service Unavailable`

```json
{
  "code": "CANCELLATION_STATUS_UNKNOWN",
  "message": "Cancellation result could not be confirmed",
  "deliveryId": "dlv-1001",
  "currentStatus": "ACCEPTED"
}
```

`currentStatus` отражает последнее подтверждённое состояние Delivery в платформе и не гарантирует, что фактическое состояние у Carrier совпадает с ним.

Платформа не устанавливает статус `CANCELLED`, пока отмена не будет подтверждена Carrier.

Для определения фактического результата запускается reconciliation — проверка состояния доставки через API перевозчика.

Повторный запрос отмены не должен создавать дублирующие операции у Carrier. Для этого используются доступные механизмы идемпотентности и проверка результата предыдущего запроса.

Checkout может получить актуальное состояние через `GET /deliveries/{deliveryId}`. До подтверждения результата отмена не должна отображаться пользователю как успешно завершённая.
---

### Отмена недопустима в текущем состоянии

Если Delivery уже передана перевозчику или находится в другом состоянии, не допускающем обычную отмену:

`409 Conflict`

```json
{
  "code": "DELIVERY_CANCELLATION_NOT_ALLOWED",
  "message": "Delivery cannot be cancelled in the current status",
  "currentStatus": "PICKED_UP"
}
```

`409 Conflict` используется, потому что запрос сам по себе корректен, но операция конфликтует с текущим состоянием ресурса.

Например:

```text
PICKED_UP
IN_TRANSIT
OUT_FOR_DELIVERY
READY_FOR_PICKUP
DELIVERED
RETURN_IN_PROGRESS
RETURNED
```

не должны переходить напрямую в `CANCELLED`.

---

### Delivery не найдена

Если `deliveryId` не существует:

`404 Not Found`

```json
{
  "code": "DELIVERY_NOT_FOUND",
  "message": "Delivery not found"
}
```

---

### Внутренняя ошибка

При непредвиденной ошибке платформы:

`500 Internal Server Error`
## 9. Запуск возврата

### POST /deliveries/{deliveryId}/return

Запускает процесс возврата Shipment, если обычная отмена Delivery уже невозможна.

Операция применяется, когда Shipment уже был передан Carrier.

Например:

```text
PICKED_UP
IN_TRANSIT
DELIVERY_FAILED
READY_FOR_PICKUP
```

### Успешный запрос

`202 Accepted`

```json
{
  "deliveryId": "dlv-1001",
  "status": "RETURN_IN_PROGRESS"
}
```

`202 Accepted` используется, поскольку физический возврат выполняется асинхронно и не завершается в момент HTTP-запроса.

После фактического возврата Shipment состояние изменяется:

```text
RETURN_IN_PROGRESS → RETURNED
```

### Возврат недопустим

Если Delivery находится в состоянии, из которого возврат не разрешён:

`409 Conflict`

```json
{
  "code": "DELIVERY_RETURN_NOT_ALLOWED",
  "message": "Return cannot be started in the current status",
  "currentStatus": "DELIVERED"
}
```

### Повторный запрос возврата

Операция должна быть идемпотентной.

Если возврат уже запущен и Delivery находится в статусе `RETURN_IN_PROGRESS`, повторный запрос не должен инициировать ещё одну операцию возврата у Carrier.

`202 Accepted`

```json
{
  "deliveryId": "dlv-1001",
  "status": "RETURN_IN_PROGRESS"
}
```

Если возврат уже завершён, платформа возвращает текущее состояние доставки.

`200 OK`

```json
{
  "deliveryId": "dlv-1001",
  "status": "RETURNED"
}
```

### Delivery не найдена

Если указанный `deliveryId` не существует:

`404 Not Found`

```json
{
  "code": "DELIVERY_NOT_FOUND",
  "message": "Delivery not found"
}
```

### Внутренняя ошибка

При непредвиденной внутренней ошибке платформы:

`500 Internal Server Error`

Повторное обращение не должно создавать дублирующие операции у внешнего Carrier.

## Общие ошибки API

Все бизнес-endpoint Delivery Orchestration Platform требуют аутентификации и проверки прав доступа вызывающего сервиса.

### 401 Unauthorized

Возвращается, если вызывающий сервис не прошёл аутентификацию: отсутствует токен доступа, токен недействителен или срок его действия истёк.

```json
{
  "code": "UNAUTHORIZED",
  "message": "Authentication required"
}
```

### 403 Forbidden

Возвращается, если вызывающий сервис успешно прошёл аутентификацию, но не имеет необходимых прав для выполнения операции.

```json
{
  "code": "FORBIDDEN",
  "message": "Insufficient permissions"
}
```

### Общие правила обработки ошибок

- `401` и `403` применяются ко всем защищённым бизнес-endpoint платформы.
- Детальная информация о токенах, внутренних настройках безопасности и причинах отказа не раскрывается в ответе.
- Попытки неавторизованного доступа должны фиксироваться в системе мониторинга безопасности.
- Формат ошибок должен быть единым для всех REST API платформы.
