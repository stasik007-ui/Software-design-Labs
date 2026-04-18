

## Part 1 — Component Diagram (30%)

**Task:** Create a Component Diagram that shows system components, their responsibilities, and interactions between them.

```mermaid
graph TD
    ClientA[Client / Sender] -->|1. Надсилає повідомлення| API[Backend API]
    ClientB[Client / Receiver] <-->|5. Підключається / Запитує дані| API
    
    API -->|2. Перевірка та маршрутизація| MsgService[Message Service]
    
    MsgService -->|3. Зберігає зі статусом 'Pending'| DB[(Messages DB)]
    MsgService -->|4. Відправляє подію| Queue[Message Queue / RabbitMQ]
    
    Queue -->|Споживає подію| Delivery[Delivery Service]
    Delivery -->|Спроба Push-повідомлення| PushNotification[Push Service / FCM]
    PushNotification -.->|Сповіщає| ClientB
    Delivery -->|Оновлює статус якщо помилка/офлайн| DB
```

---

## Part 2 — Sequence Diagram (25%)

**Scenario:** User A sends a message to user B who is offline.
**Task:** Describe the interaction sequence in time.

```mermaid
sequenceDiagram
    actor UserA
    participant ClientA
    participant API
    participant MsgService as Message Service
    participant DB
    participant Queue
    participant Delivery
    actor UserB

    UserA->>ClientA: Пише повідомлення
    ClientA->>API: POST /messages
    API->>MsgService: processMessage()
    
    MsgService->>DB: save(msg, status="Sent")
    MsgService->>Queue: enqueue(msg_id)
    API-->>ClientA: 202 Accepted (Збережено на сервері)

    Queue->>Delivery: consume(msg_id)
    Delivery->>UserB: Спроба доставити (Push / WebSocket)
    note right of Delivery: User B is OFFLINE
    Delivery--xUserB: Доставка не вдалася (Timeout)
    Delivery->>DB: updateStatus(msg_id, "Pending")

    note over UserB, API: Минає час... User B з'являється в мережі
    
    UserB->>ClientB: Відкриває додаток
    ClientB->>API: GET /messages/sync (Pull/Sync)
    API->>MsgService: getPendingMessages(user_id=B)
    MsgService->>DB: fetch(status="Pending", receiver=B)
    DB-->>MsgService: Список повідомлень
    MsgService-->>API: Дані
    API-->>ClientB: 200 OK + Повідомлення
    ClientB->>UserB: Відображає повідомлення
    
    ClientB->>API: POST /messages/ack (Підтвердження)
    API->>DB: updateStatus(msg_ids, "Delivered")
```

---

## Part 3 — State Diagram (20%)

**Object:** `Message`
**Task:** Describe the message lifecycle.

```mermaid
stateDiagram-v2
    [*] --> Created: Користувач натиснув "Відправити"
    
    Created --> Sent: Збережено на сервері (1 галочка)
    
    Sent --> Pending: Отримувач офлайн (спроба Push не вдалася)
    Sent --> Delivered: Отримувач онлайн (успішна доставка)
    
    Pending --> Retrying: Сервер робить повторну спробу
    Retrying --> Pending: Знову невдача (все ще офлайн)
    Retrying --> Failed: Досягнуто ліміт спроб Push
    
    Pending --> Delivered: Клієнт синхронізував дані при вході
    Failed --> Delivered: Клієнт синхронізував дані при вході
    
    Delivered --> Read: Користувач відкрив чат (2 сині галочки)
    
    Read --> [*]
```

---

## Part 4 — ADR (Architecture Decision Record) (25%)

**Task:** Document one architecture decision.

```markdown
# ADR-001: Use Hybrid Delivery (Push + Pull Sync) for Offline Users

## Status
Accepted

## Context
Our system must support users who are offline for extended periods. The challenge is ensuring messages are not lost while balancing server load. Using only a Message Queue with infinite retries is resource-intensive, and using only Client Polling drains client battery.

## Decision
We will use a hybrid approach:
1. Store all messages persistently in a Database immediately upon receipt.
2. Use a Message Queue and Delivery Service to attempt initial Push notifications.
3. If delivery fails (user is offline), mark the message as `Pending` in the DB and remove it from the active retry queue.
4. Require the client application to perform a "Sync" (Pull request) against the Backend API whenever it comes back online to retrieve all `Pending` messages.

## Alternatives
- **Infinite Queue Retry (Rejected):** Storing messages in a queue like RabbitMQ/Kafka for weeks until a user connects is inefficient and risks data loss if the queue is purged.
- **Strict Client Polling (Rejected):** Forcing clients to ping the server every 5 seconds drains battery and creates unnecessary network traffic.

## Consequences
+ **Positive:** 100% reliable delivery; no messages are lost even during long offline periods. Reduced load on message brokers.
- **Negative:** Increased complexity in the client-side application (must implement synchronization logic on startup). Increased load on the database during mass reconnect events.
```