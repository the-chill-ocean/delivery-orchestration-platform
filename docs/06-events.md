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

1. проверяет событие на повторную обработку;
2. находит выбранный `DeliveryOffer` и snapshot `Shipment`;
3. создаёт внутреннюю сущность `Delivery` и записывает `DeliveryCreated` в Transactional Outbox в рамках одной транзакции;
4. определяет подходящего Carrier;
5. создаёт `DeliveryAttempt` с указанием `carrierId`;
6. отправляет запрос на создание доставки через соответствующий Carrier Adapter;
7. сохраняет `carrierDeliveryId` после подтверждения перевозчика;
8. изменяет состояние `Delivery` на `ACCEPTED`.
   
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

Delivery Orchestration Platform получает согласованные условия доставки из ранее сохранённого `DeliveryOffer`.

Физические характеристики отправления, точку отправления и адрес назначения платформа получает из локального snapshot `Shipment`.

Таким образом, событие `OrderReadyForDelivery` содержит только идентификаторы, необходимые для запуска создания доставки. Параметры отправления и условия доставки в событии не дублируются.
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

# 10. Согласование альтернативных условий доставки

Если Carrier отказался выполнять доставку, Delivery Orchestration Platform ищет альтернативного перевозчика.

Если для курьерской доставки альтернативный Carrier сохраняет согласованные условия, переключение выполняется автоматически.

Если изменяются условия доставки или требуется выбрать новый ПВЗ, необходимо подтверждение пользователя.

### DeliveryAlternativeApprovalRequired

Delivery Orchestration Platform публикует событие с предложением альтернативного варианта.

**Producer:** Delivery Orchestration Platform

**Consumer:** Order Service

```json
{
  "eventId": "evt-10004",
  "eventType": "DeliveryAlternativeApprovalRequired",
  "occurredAt": "2026-09-18T17:00:00+04:00",
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "alternativeOfferId": "off-1003"
}
```

Order Service организует взаимодействие с пользователем и согласование изменившихся условий.

Если изменяется стоимость, Order Service также координирует необходимые действия с Payment Service.


### DeliveryAlternativeApproved

После подтверждения пользователем новых условий и выполнения необходимых платёжных операций Order Service публикует событие `DeliveryAlternativeApproved`.

После получения события Delivery Orchestration Platform:

1. Проверяет событие на повторную обработку.
2. Находит соответствующие `Delivery` и альтернативный `DeliveryOffer`.
3. Проверяет, что подтверждение относится к ожидаемому альтернативному предложению.
4. Обновляет согласованные условия в `Delivery`:
   - `deliveryOfferId`;
   - `price`;
   - `estimatedDeliveryFrom`;
   - `estimatedDeliveryTo`;
   - `pickupPointId`, если применяется доставка в ПВЗ.
5. Определяет Carrier на основании подтверждённого предложения.
6. Создаёт новый `DeliveryAttempt` с указанием `carrierId`.
7. Отправляет запрос на создание доставки через соответствующий Carrier Adapter.

Обновление согласованных условий Delivery должно выполняться атомарно.

История предыдущих попыток доставки сохраняется в `DeliveryAttempt`. Исходный `DeliveryOffer` также сохраняется для аудита.

### DeliveryAlternativeRejected

Если пользователь отказывается от альтернативного предложения, Order Service публикует `DeliveryAlternativeRejected`.

Delivery Orchestration Platform прекращает попытки оформления альтернативной доставки и завершает или отменяет Delivery в соответствии с её текущим состоянием.

Order Service отвечает за отмену заказа и организацию возврата денежных средств, если это предусмотрено бизнес-сценарием.

Повторная обработка событий подтверждения и отказа должна быть идемпотентной.
---
# 11. Разделение ответственности

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
