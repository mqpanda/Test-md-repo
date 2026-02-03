## Схема таблиц (Prisma-like)

```prisma
model User {
  id String @id @default(uuid())
  email String? @unique
  password String
  emailVerifiedAt DateTime?
  deletedAt DateTime?
  subscriptions Subscription[]
  payments Payment[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
}

model Subscription {
  id String @id @default(uuid())
  userId String @unique
  status SubscriptionStatus
  currentPeriodStart DateTime
  currentPeriodEnd   DateTime
  cancelAtPeriodEnd Boolean @default(false)
  canceledAt DateTime?
  user User @relation(fields: [userId], references: [id])
  payments Payment[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([status])
  @@index([currentPeriodEnd])
}

model Payment {
  id String @id @default(uuid())
  provider String
  externalPaymentId String
  userId String
  subscriptionId String?
  amount Decimal @db.Decimal(18,2)
  currency String
  status PaymentStatus
  paidAt DateTime?
  metadata Json?
  user User @relation(fields: [userId], references: [id])
  subscription Subscription? @relation(fields: [subscriptionId], references: [id])
  createdAt DateTime @default(now())

  @@unique([provider, externalPaymentId])
  @@index([userId])
  @@index([subscriptionId])
  @@index([status])
}

model WebhookEvent {
  id String @id @default(uuid())
  provider String
  eventId String
  eventType String
  externalPaymentId String?
  payload Json
  status WebhookEventStatus
  error String?
  receivedAt DateTime @default(now())
  processedAt DateTime?

  @@unique([provider, eventId])
  @@index([status])
  @@index([externalPaymentId])
}

enum SubscriptionStatus { ACTIVE CANCELED EXPIRED PAST_DUE }
enum PaymentStatus { PENDING SUCCEEDED FAILED REFUNDED }
enum WebhookEventStatus { RECEIVED PROCESSED FAILED IGNORED }
```

**Ключевые уникальности и индексы:**

- `User.email` — уникальный, индекс для поиска
- `Subscription.userId` — уникальный (1 подписка на пользователя)
- `Payment.provider + externalPaymentId` — уникальный
- `WebhookEvent.provider + eventId` — уникальный, воимя избежания двойной обработки
- Индексы на `status`, `currentPeriodEnd` для быстрого поиска активных или истекающих подписок

---

## Псевдокод обработчика webhook

```
POST /webhook

// Валидация входа
if request.rawBody empty or required fields missing:
    return 400
payload = parseJSON(request.rawBody)

// Проверка подписи
signature = headers["x-provider-signature"]
if !verifySignature(rawBody, signature, providerSecret):
    return 401

// Дедупликация
try:
    webhookEvent = db.webhookEvent.create({
        provider, eventId, eventType, externalPaymentId,
        payload, status: RECEIVED
    })
catch UniqueViolation:
    return 200  // webhook уже обработан

// Обработка в транзакции
try:
    db.transaction(tx => {

        event = tx.webhookEvent.findUnique(eventId)
        if event.status == PROCESSED: return

        existingPayment = tx.payment.findUnique(provider + externalPaymentId)
        if existingPayment exists:
            tx.webhookEvent.update({status: PROCESSED, processedAt: now()})
            return

        user = tx.user.findByEmail(payload.email)
        if !user: throw BusinessError("User not found")

        payment = tx.payment.create({provider, externalPaymentId, userId, amount, currency, status: SUCCEEDED, paidAt: now()})

        subscription = tx.subscription.findUnique(userId)
        if !subscription:
            subscription = tx.subscription.create({userId, status: ACTIVE, currentPeriodStart: now(), currentPeriodEnd: now()+planDuration})
        else if subscription.status == ACTIVE:
            baseDate = max(now(), subscription.currentPeriodEnd)
            subscription = tx.subscription.update({currentPeriodEnd: baseDate+planDuration})
        else:
            subscription = tx.subscription.update({status: ACTIVE, currentPeriodStart: now(), currentPeriodEnd: now()+planDuration, cancelAtPeriodEnd: false, canceledAt: null})

        tx.payment.update({id: payment.id, subscriptionId: subscription.id})
        tx.webhookEvent.update({status: PROCESSED, processedAt: now()})
    })
catch err:
    db.webhookEvent.update({status: FAILED, error: err.message})
    return 500

return 200
```

**HTTP-коды:**

- `200 OK` — успешно или повторное событие
- `400` — битый payload / неизвестный provider
- `401` — подпись невалидна
- `500` — ошибка транзакции / падение сервера

---

## Edge Cases

| Сценарий                                   | Обработка                                                                        |
| ------------------------------------------ | -------------------------------------------------------------------------------- |
| Webhook пришел дважды                      | UNIQUE constraint → 200 OK, идемпотентно                                         |
| Webhook пришел раньше user                 | Сохраняем webhook со статусом `FAILED` / `RECEIVED`, ждём создания user          |
| Webhook без email, есть externalPaymentId  | Дедупликация по externalPaymentId, логируем, не создаём payment/subscription     |
| Webhook с другой суммой                    | Логируем `amount_mismatch`, ставим на manual review, не активируем автоматически |
| Webhook через неделю                       | Безопасно активируем/продлеваем subscription, учитывая `currentPeriodEnd`        |
| Сервер упал после payment, до subscription | При retry event обработается заново; транзакция гарантирует согласованность      |

---

## Debuggability и наблюдаемость

### Логи (обязательные поля)

**Webhook:**

- provider
- eventId
- eventType
- externalPaymentId
- receivedAt
- payload (маскировать чувствительные данные)
- headers (только ключевые, signature)

**Дедупликация / обработка:**

- webhookEvent.status (`RECEIVED`/`PROCESSED`/`FAILED`)
- userId / subscriptionId / paymentId
- ошибки unique / транзакции
- timestamps начала и конца транзакции

**Ошибки / исключения:**

- stack trace / message
- snapshot payload
- retryCount

---

### Метрики

- `webhook.received_total`
- `webhook.processed_total`
- `webhook.failed_total`
- `payment.created_total`
- `subscription.activated_total` / `subscription.extended_total`
- `webhook.processing_duration_seconds`

---

### Алерты

- Webhook failures > threshold (например 5 за 5 минут)
- Duplicate payments / conflicts
- Payment amount mismatch
- Subscription extension failed
- Webhook queue backlog
