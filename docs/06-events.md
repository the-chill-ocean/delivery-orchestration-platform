# События и асинхронные взаимодействия

Документ описывает основные события Delivery Orchestration Platform и взаимодействие через Kafka.

Асинхронное взаимодействие используется там, где вызывающей системе не требуется немедленный результат операции и важно снизить связанность между сервисами.

---

## 1. Общая схема

```text
Order Service
    |
    | OrderReadyForDelivery
    v
Kafka
    |
    v
Delivery Orchestration Platform
    |
    +--> создание Delivery
    |
    +--> создание DeliveryAttempt
    |
    v
Carrier API
```

После создания и изменения состояния доставки Delivery Orchestration Platform публикует события для заинтересованных систем:

```text
Carrier
    |
    | webhook / API response
    v
Delivery Orchestration Platform
    |
    | нормализация статуса
    v
Kafka
    |
    +--> Order Service
    +--> Notification Service
    +--> другие consumers
```

---

# 2. OrderReadyForDelivery

Событие сообщает Delivery Orchestration Platform, что конкретный Shipment готов к передаче в доставку.

### Producer

`Order Service`

### Consumer

`Delivery Orchestration Platform`

### Назначение

После получения события платформа:

1. находит выбранный `DeliveryOffer`;
2. создаёт `Delivery`;
3. создаёт первый `DeliveryAttempt`;
4. определяет Carrier;
5. отправляет запрос на создание доставки во внешнюю систему перевозчика.

### Пример события

```json
{
  "eventId": "evt-10001",
  "eventType": "OrderReadyForDelivery",
  "occurredAt": "2026-09-18T12:30:00+04:00",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "deliveryOfferId": "off-1001"
}
```

### Поля события

| Поле | Описание |
|---|---|
| `eventId` | Уникальный идентификатор события |
| `eventType` | Тип события |
| `occurredAt` | Дата и время возникновения события |
| `orderId` | Идентификатор заказа |
| `shipmentId` | Идентификатор отправления |
| `deliveryOfferId` | Идентификатор выбранного предложения доставки |

Delivery Orchestration Platform получает параметры доставки из ранее сохранённого `DeliveryOffer`.

Поэтому в событии не требуется дублировать стоимость, сроки, Carrier и параметры Shipment.

---

# 3. DeliveryCreated

Событие публикуется после успешного создания внутренней сущности `Delivery`.

### Producer

`Delivery Orchestration Platform`

### Consumers

Например:

- `Order Service`;
- другие внутренние системы, которым необходимо знать идентификатор Delivery.

### Пример события

```json
{
  "eventId": "evt-10002",
  "eventType": "DeliveryCreated",
  "occurredAt": "2026-09-18T12:30:05+04:00",
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "deliveryType": "COURIER",
  "status": "CREATED"
}
```

### Поля события

| Поле | Описание |
|---|---|
| `eventId` | Уникальный идентификатор события |
| `eventType` | Тип события |
| `occurredAt` | Дата и время возникновения события |
| `deliveryId` | Идентификатор Delivery |
| `orderId` | Идентификатор заказа |
| `shipmentId` | Идентификатор отправления |
| `deliveryType` | Тип доставки: `COURIER` или `PICKUP_POINT` |
| `status` | Текущее состояние Delivery |

---

# 4. DeliveryStatusChanged

Событие публикуется при изменении нормализованного состояния Delivery.

Внешние статусы Carrier не публикуются во внутренние системы напрямую.

Delivery Orchestration Platform сначала преобразует их во внутреннюю модель состояний.

Например:

```text
Carrier status: courier_received_package

                    ↓ normalization

Delivery status: PICKED_UP
```

### Producer

`Delivery Orchestration Platform`

### Возможные consumers

- `Order Service`;
- `Notification Service`;
- системы клиентского интерфейса;
- аналитические системы.

### Пример события

```json
{
  "eventId": "evt-10003",
  "eventType": "DeliveryStatusChanged",
  "occurredAt": "2026-09-18T16:10:00+04:00",
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "previousStatus": "PICKED_UP",
  "status": "IN_TRANSIT"
}
```

### Поля события

| Поле | Описание |
|---|---|
| `eventId` | Уникальный идентификатор события |
| `eventType` | Тип события |
| `occurredAt` | Дата и время изменения состояния |
| `deliveryId` | Идентификатор Delivery |
| `orderId` | Идентификатор заказа |
| `shipmentId` | Идентификатор отправления |
| `previousStatus` | Предыдущее состояние Delivery |
| `status` | Новое состояние Delivery |

---

# 5. Получение статусов от Carrier

Способ получения состояния зависит от возможностей конкретного перевозчика.

Предпочтительный вариант:

```text
Carrier
    |
    | webhook
    v
Delivery Orchestration Platform
```

Если Carrier не поддерживает webhook, может использоваться периодический запрос его API.

Полученный внешний статус:

1. идентифицируется по `carrierDeliveryId`;
2. сопоставляется с соответствующим `DeliveryAttempt`;
3. преобразуется во внутренний статус Delivery;
4. проверяется допустимость перехода;
5. сохраняется новое состояние;
6. при изменении состояния публикуется `DeliveryStatusChanged`.

---

# 6. Нормализация статусов

Каждый Carrier может использовать собственную модель состояний.

Например:

```text
Carrier A:
CREATED
AT_SORTING_CENTER
LEFT_SORTING_CENTER
WITH_COURIER
COMPLETED

Carrier B:
NEW
PROCESSING
TRANSPORTATION
LAST_MILE
DONE
```

Delivery Orchestration Platform преобразует их в единую внутреннюю модель:

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

Таким образом внутренние consumers не зависят от особенностей API конкретного Carrier.

---

# 7. Kafka key

Для событий жизненного цикла Delivery в качестве Kafka key используется:

```text
deliveryId
```

Это позволяет событиям одной Delivery попадать в одну partition и сохранять порядок обработки в рамках этой доставки.

Для `OrderReadyForDelivery`, когда `deliveryId` ещё не существует, используется:

```text
shipmentId
```

---

# 8. Общие поля событий

Все события должны содержать минимум:

```json
{
  "eventId": "evt-...",
  "eventType": "...",
  "occurredAt": "..."
}
```

### eventId

Уникальный идентификатор события.

Используется в том числе для защиты consumers от повторной обработки одного и того же сообщения.

### eventType

Тип события.

Например:

```text
OrderReadyForDelivery
DeliveryCreated
DeliveryStatusChanged
```

### occurredAt

Время возникновения бизнес-события.

Формат:

```text
ISO 8601
```

Пример:

```text
2026-09-18T16:10:00+04:00
```

---

# 9. Гарантии доставки сообщений

Delivery Orchestration Platform не предполагает, что Kafka доставит каждое сообщение строго один раз.

Consumer должен быть готов к повторному получению одного события.

Например:

```text
OrderReadyForDelivery
OrderReadyForDelivery
```

не должно приводить к созданию двух Delivery для одного Shipment.

Механизм идемпотентной обработки, retry и DLQ описывается отдельно в:

```text
docs/07-reliability.md
```

---

# 10. Разделение ответственности

### Order Service

Отвечает за:

- жизненный цикл Order;
- определение момента готовности Shipment к доставке;
- публикацию `OrderReadyForDelivery`.

### Delivery Orchestration Platform

Отвечает за:

- создание Delivery;
- создание DeliveryAttempt;
- взаимодействие с Carrier;
- нормализацию статусов;
- изменение жизненного цикла Delivery;
- публикацию событий о Delivery.

### Carrier

Отвечает за:

- фактическое выполнение доставки;
- внешний идентификатор доставки;
- собственную модель состояний;
- передачу информации об изменении состояния через webhook или API.
