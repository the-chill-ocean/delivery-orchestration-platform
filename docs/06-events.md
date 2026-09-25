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

## 2. OrderReadyForDelivery

Событие сообщает Delivery Orchestration Platform, что конкретный Shipment готов к передаче в доставку.

### Producer

`Order Service`

### Consumer

`Delivery Orchestration Platform`

### Назначение

После получения события платформа:

1. Проверяет событие на повторную обработку.
2. Находит выбранный DeliveryOffer и snapshot Shipment.
3. В одной транзакции создаёт Delivery, записывает DeliveryCreated в Transactional Outbox и сохраняет задание на оформление доставки.
4. После успешного commit подтверждает обработку Kafka-сообщения.
5. Фоновый обработчик получает сохранённое задание, определяет Carrier и создаёт DeliveryAttempt.
6. Через соответствующий Carrier Adapter отправляет запрос на создание доставки.
7. После подтверждения Carrier сохраняет carrierDeliveryId и изменяет статус Delivery на ACCEPTED.

Незавершённые задания сохраняются в БД и могут быть продолжены после перезапуска приложения.

### Пример события

```json
{
  "eventId": "evt-10001",
  "eventType": "OrderReadyForDelivery",
  "occurredAt": "2026-09-24T12:30:00+04:00",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "deliveryOfferId": "off-1001",
  "offerConfirmedAt": "2026-09-24T11:15:00+04:00"
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
| `offerConfirmedAt` | Дата и время подтверждения пользователем выбранного DeliveryOffer, зафиксированные Order Service |

### Проверка подтверждённого предложения

Order Service публикует `OrderReadyForDelivery` только для Shipment, у которого есть подтверждённое пользователем предложение доставки и выполнены необходимые условия оформления заказа.

При получении события Delivery Orchestration Platform:

1. Находит сохранённый DeliveryOffer по `deliveryOfferId`.
2. Проверяет принадлежность предложения указанному `shipmentId`.
3. Проверяет, что `offerConfirmedAt` не превышает `validUntil` соответствующего предложения.
4. Использует сохранённые условия DeliveryOffer для создания Delivery.

Если проверка не пройдена, платформа не создаёт Delivery. Ошибка фиксируется для последующего разбора.

Истечение `validUntil` после подтверждения предложения не отменяет согласованные условия доставки.

Delivery Orchestration Platform получает согласованные условия доставки из ранее сохранённого `DeliveryOffer`.

Физические характеристики отправления, точку отправления и адрес назначения платформа получает из локального snapshot `Shipment`.

Таким образом, событие `OrderReadyForDelivery` содержит идентификаторы и метаданные подтверждения, необходимые для запуска создания доставки.

### Связывание Shipment с Order

При обработке `OrderReadyForDelivery` платформа находит ранее сохранённый snapshot Shipment по `shipmentId`.

Если `orderId` ещё не заполнен, платформа сохраняет идентификатор заказа из события.

Если Shipment уже связан с другим `orderId`, событие не должно приводить к созданию Delivery. Несоответствие фиксируется для расследования.

После проверки связи платформа продолжает создание Delivery.
---

## 3. DeliveryCreated

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

## 4. DeliveryStatusChanged

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

## 5. Получение статусов от Carrier

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

## 6. Нормализация статусов

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

## 7. Kafka key

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

## 8. Общие поля событий

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

## 9. Гарантии доставки сообщений

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

## 10. Согласование альтернативных условий доставки

Если Carrier отказался выполнять доставку, Delivery Orchestration Platform ищет альтернативного перевозчика.

Если для курьерской доставки альтернативный Carrier сохраняет согласованные условия, переключение выполняется автоматически.

Если изменяются условия доставки или требуется выбрать новый ПВЗ, необходимо подтверждение пользователя.

### DeliveryAlternativeApprovalRequired

Delivery Orchestration Platform публикует событие с предложением альтернативного варианта.

**Producer:** Delivery Orchestration Platform

**Consumer:** Order Service

**Kafka key:** `deliveryId`

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
После получения `DeliveryAlternativeApprovalRequired` Order Service использует `alternativeOfferId` для получения актуальных условий предложения через:

```text
POST /delivery-offers/{alternativeOfferId}/validate
```
Результат повторной проверки используется для отображения пользователю актуальной стоимости, срока доставки и ПВЗ, если применимо.

Если результат — `VALID`, для согласования используется исходный `alternativeOfferId`.

Если результат — `CHANGED`, пользователю показываются условия из `currentOffer`, а при последующем подтверждении используется новый `currentOffer.offerId`.

Если результат — `UNAVAILABLE`, исходное альтернативное предложение больше не может быть подтверждено и платформа должна подобрать другой вариант.

Order Service организует взаимодействие с пользователем и согласование изменившихся условий.

Если изменяется стоимость, Order Service также координирует необходимые действия с Payment Service.


### DeliveryAlternativeApproved

После подтверждения пользователем новых условий и выполнения необходимых платёжных операций Order Service публикует событие `DeliveryAlternativeApproved`.

После получения события Delivery Orchestration Platform:

1. Проверяет событие на повторную обработку.
2. Находит соответствующие Delivery и альтернативный DeliveryOffer.
3. Проверяет, что подтверждение относится к ожидаемому альтернативному предложению.
4. В рамках одной транзакции БД:
   - регистрирует `eventId` в таблице `processed_events`;
   - обновляет согласованные параметры Delivery: `deliveryOfferId`, `price`, `currency`, `estimatedDeliveryFrom`, `estimatedDeliveryTo` и `pickupPointId`, если применимо;
   - создаёт внутреннее задание на повторное оформление доставки со статусом `PENDING`.
5. Фиксирует транзакцию и подтверждает обработку Kafka-сообщения.
6. Фоновый обработчик получает задание, определяет Carrier и создаёт новый DeliveryAttempt.
7. Отправляет запрос на создание доставки через соответствующий Carrier Adapter.

Регистрация обработанного события, изменение Delivery и сохранение внутреннего задания выполняются атомарно.

Если приложение завершится после подтверждения Kafka-сообщения, незавершённое задание останется в БД и сможет быть обработано повторно.

Повторное получение события не должно приводить к созданию дополнительных заданий или DeliveryAttempt для одного подтверждённого предложения.

История предыдущих DeliveryAttempt и исходный DeliveryOffer сохраняются для аудита.

#### Формат события

**Producer:** Order Service  
**Consumer:** Delivery Orchestration Platform  
**Kafka key:** `deliveryId`

```json
{
  "eventId": "evt-10005",
  "eventType": "DeliveryAlternativeApproved",
  "occurredAt": "2026-09-18T17:15:00+04:00",
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "alternativeOfferId": "off-1003"
}
```

Событие публикуется после подтверждения пользователем альтернативного предложения и успешного выполнения необходимых платёжных операций.

### DeliveryAlternativeRejected

Если пользователь отказывается от альтернативного предложения, Order Service публикует `DeliveryAlternativeRejected`.

Delivery Orchestration Platform прекращает попытки оформления альтернативной доставки и завершает или отменяет Delivery в соответствии с её текущим состоянием.

Order Service отвечает за отмену заказа и организацию возврата денежных средств, если это предусмотрено бизнес-сценарием.

Повторная обработка событий подтверждения и отказа должна быть идемпотентной.

#### Формат события

**Producer:** Order Service  
**Consumer:** Delivery Orchestration Platform  
**Kafka key:** `deliveryId`

```json
{
  "eventId": "evt-10006",
  "eventType": "DeliveryAlternativeRejected",
  "occurredAt": "2026-09-18T17:15:00+04:00",
  "deliveryId": "dlv-1001",
  "orderId": "ord-501",
  "shipmentId": "shp-501",
  "alternativeOfferId": "off-1003"
}
```

Событие сообщает, что пользователь отказался от конкретного альтернативного предложения.

Delivery Orchestration Platform не создаёт новый DeliveryAttempt на основании отклонённого предложения. Дальнейшие действия определяются текущим состоянием Delivery и согласованным бизнес-сценарием.

### Поля событий согласования

| Поле | Описание |
|---|---|
| `eventId` | Уникальный идентификатор события |
| `eventType` | Тип события: `DeliveryAlternativeApprovalRequired`, `DeliveryAlternativeApproved` или `DeliveryAlternativeRejected` |
| `occurredAt` | Дата и время возникновения события |
| `deliveryId` | Идентификатор доставки, для которой предложены альтернативные условия |
| `orderId` | Идентификатор связанного заказа |
| `shipmentId` | Идентификатор отправления |
| `alternativeOfferId` | Идентификатор альтернативного предложения. В `DeliveryAlternativeApprovalRequired` — предложение для согласования; в `DeliveryAlternativeApproved` и `DeliveryAlternativeRejected` — предложение, по которому пользователь принял решение |
---
## 11. Разделение ответственности

### Order Service

Отвечает за:

- жизненный цикл Order;
- определение момента готовности Shipment к доставке;
- публикацию события `OrderReadyForDelivery`;
- получение события `DeliveryAlternativeApprovalRequired`;
- организацию согласования изменившихся условий доставки с пользователем через соответствующие системы;
- координацию необходимых операций с Payment Service при изменении стоимости доставки;
- публикацию `DeliveryAlternativeApproved` после подтверждения новых условий и завершения необходимых платёжных операций;
- публикацию `DeliveryAlternativeRejected` при отказе пользователя;
- получение событий о создании и изменении состояния Delivery.

### Delivery Orchestration Platform

Отвечает за:

- расчёт и повторную проверку DeliveryOffer;
- создание и управление Delivery;
- создание и управление DeliveryAttempt;
- выбор Carrier и взаимодействие с его API;
- поиск альтернативных вариантов доставки;
- публикацию `DeliveryAlternativeApprovalRequired`, если необходимо согласование новых условий;
- обработку событий `DeliveryAlternativeApproved` и `DeliveryAlternativeRejected`;
- нормализацию внешних статусов Carrier;
- управление жизненным циклом Delivery;
- публикацию `DeliveryCreated` и `DeliveryStatusChanged`.

### Payment Service

Отвечает за выполнение платёжных операций.

При изменении стоимости доставки необходимые списания или возвраты координируются Order Service.

Delivery Orchestration Platform не управляет платежами напрямую.

### Carrier

Отвечает за:

- расчёт тарифов и сроков доставки;
- фактическое выполнение доставки;
- создание доставки во внешней системе;
- обработку запросов на отмену и возврат;
- предоставление внешнего идентификатора `carrierDeliveryId`;
- передачу информации об изменении состояния через webhook или API.
