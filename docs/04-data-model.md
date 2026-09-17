## 9. Логическая модель данных

Логическая модель отражает основные сущности Delivery Orchestration Platform и связи между ними.

`Order` является внешней сущностью. Владельцем заказа остаётся Order Service.

```mermaid
erDiagram

    ORDER ||--o{ SHIPMENT : contains

    SHIPMENT ||--o{ DELIVERY_OFFER : has
    SHIPMENT ||--o| DELIVERY : delivered_by

    DELIVERY_OFFER ||--o| DELIVERY : selected_for

    DELIVERY ||--o{ DELIVERY_ATTEMPT : has

    CARRIER ||--o{ DELIVERY_OFFER : provides
    CARRIER ||--o{ PICKUP_POINT : owns
    CARRIER ||--o{ DELIVERY_ATTEMPT : performs

    PICKUP_POINT o|--o{ DELIVERY_OFFER : used_in
    PICKUP_POINT o|--o{ DELIVERY : selected_for

    ORDER {
        string orderId PK
    }

    SHIPMENT {
        string shipmentId PK
        string orderId FK
        string originLocationId
        decimal weight
        string dimensions
        string contentType
        decimal declaredValue
    }

    DELIVERY_OFFER {
        string offerId PK
        string shipmentId FK
        string carrierId FK
        string pickupPointId FK
        string deliveryType
        decimal price
        string currency
        datetime estimatedDeliveryFrom
        datetime estimatedDeliveryTo
        datetime validUntil
    }

    DELIVERY {
        string deliveryId PK
        string orderId
        string shipmentId FK
        string deliveryOfferId FK
        string pickupPointId FK
        string deliveryType
        string status
        decimal price
        datetime estimatedDeliveryFrom
        datetime estimatedDeliveryTo
        datetime createdAt
        datetime updatedAt
    }

    DELIVERY_ATTEMPT {
        string deliveryAttemptId PK
        string deliveryId FK
        string carrierId FK
        string carrierDeliveryId
        string status
        string failureReason
        datetime createdAt
        datetime updatedAt
    }

    CARRIER {
        string carrierId PK
        string name
        string status
        string supportedDeliveryTypes
        string supportedRegions
        decimal maxWeight
        string maxDimensions
        string supportedContentTypes
        decimal maxDeclaredValue
    }

    PICKUP_POINT {
        string pickupPointId PK
        string carrierId FK
        string externalPickupPointId
        string address
        decimal latitude
        decimal longitude
        string status
        string workingHours
        decimal maxWeight
        string maxDimensions
    }
```
