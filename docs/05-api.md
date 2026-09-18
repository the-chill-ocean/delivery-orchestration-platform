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

---

## 2. Успешный ответ

### 200 OK

```json
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
  "estimatedDeliveryFrom": "2026-09-21",
  "estimatedDeliveryTo": "2026-09-22",
  "validUntil": "2026-09-17T21:00:00+04:00"
}
```

### Поля DeliveryOffer

| Поле | Описание |
|---|---|
| `shipmentId` | Идентификатор отправления, для которого выполнен расчёт |
| `offerId` | Идентификатор рассчитанного предложения |
| `deliveryType` | Способ доставки: `COURIER` или `PICKUP_POINT` |
| `pickupPoint` | Краткая информация о ПВЗ. Заполняется для доставки в пункт выдачи |
| `pickupPoint.pickupPointId` | Идентификатор ПВЗ |
| `pickupPoint.name` | Наименование ПВЗ |
| `pickupPoint.address` | Адрес ПВЗ |
| `pickupPoint.latitude` | Географическая широта ПВЗ |
| `pickupPoint.longitude` | Географическая долгота ПВЗ |
| `pickupPoint.workingHours` | Режим работы ПВЗ |
| `price` | Стоимость доставки |
| `price.amount` | Сумма стоимости доставки |
| `price.currency` | Валюта стоимости доставки |
| `estimatedDeliveryFrom` | Минимальная ожидаемая дата доставки |
| `estimatedDeliveryTo` | Максимальная ожидаемая дата доставки |
| `validUntil` | Дата и время, до которых предложение считается актуальным |

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

### Offer актуален

`200 OK`

```json
{
  "status": "VALID",
  "offer": {
    "offerId": "off-1001",
    "deliveryType": "COURIER",
    "price": {
      "amount": 450,
      "currency": "RUB"
    },
    "estimatedDeliveryFrom": "2026-09-20",
    "estimatedDeliveryTo": "2026-09-21",
    "validUntil": "2026-09-17T20:00:00+04:00"
  }
}
```
| Поле | Описание |
|---|---|
| `offer` | Актуальное предложение доставки |
| `offer.offerId` | Идентификатор предложения |
| `offer.deliveryType` | Способ доставки: `COURIER` или `PICKUP_POINT` |
| `offer.pickupPoint` | Информация о ПВЗ. Заполняется для доставки в пункт выдачи |
| `offer.price` | Стоимость доставки |
| `offer.price.amount` | Сумма стоимости доставки |
| `offer.price.currency` | Валюта |
| `offer.estimatedDeliveryFrom` | Минимальная ожидаемая дата доставки |
| `offer.estimatedDeliveryTo` | Максимальная ожидаемая дата доставки |
| `offer.validUntil` | Срок актуальности предложения |
### Условия изменились
`200 OK`

```json
{
  "status": "CHANGED",
  "previousOfferId": "off-1001",
  "updatedOffer": {
    "offerId": "off-1002",
    "deliveryType": "COURIER",
    "price": {
      "amount": 500,
      "currency": "RUB"
    },
    "estimatedDeliveryFrom": "2026-09-21",
    "estimatedDeliveryTo": "2026-09-22",
    "validUntil": "2026-09-17T20:00:00+04:00"
  }
}
```
Пользователь должен подтвердить обновлённые условия перед оплатой.
| Поле | Описание |
|---|---|
| `previousOfferId` | Идентификатор предложения, условия которого изменились |
| `updatedOffer` | Новое предложение с актуальными условиями |
| `updatedOffer.offerId` | Идентификатор нового предложения |
| `updatedOffer.deliveryType` | Способ доставки |
| `updatedOffer.pickupPoint` | Информация о ПВЗ, если применимо |
| `updatedOffer.price` | Новая стоимость доставки |
| `updatedOffer.price.amount` | Новая сумма стоимости доставки |
| `updatedOffer.price.currency` | Валюта |
| `updatedOffer.estimatedDeliveryFrom` | Новая минимальная дата доставки |
| `updatedOffer.estimatedDeliveryTo` | Новая максимальная дата доставки |
| `updatedOffer.validUntil` | Срок актуальности нового предложения |

`validUntil` определяет срок, до которого DeliveryOffer может быть выбран или подтверждён пользователем.

После успешной проверки и подтверждения предложения перед оплатой истечение `validUntil` не удаляет DeliveryOffer и не отменяет ранее согласованные условия.

DeliveryOffer сохраняется как историческая запись и используется при последующем создании Delivery.
### Offer больше недоступен
`200 OK`

```json
{
  "status": "UNAVAILABLE",
  "offerId": "off-1001"
}
```
| Поле | Описание |
|---|---|
| `offerId` | Идентификатор предложения, которое больше недоступно |
В этом случае Checkout должен запросить новые варианты доставки.
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
