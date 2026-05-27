In system design, a **Saga** is a pattern used to manage **distributed transactions** across multiple services.

Instead of doing one giant database transaction across services (which is slow, fragile, and often impossible), a saga breaks the workflow into **small local transactions**.

Each service:

1. Does its own database transaction
2. Publishes an event or calls the next service
3. If something fails later, previous steps are **compensated** (undone)

---

# Why Saga Exists

Suppose you have an e-commerce system:

1. Order Service → create order
2. Payment Service → charge user
3. Inventory Service → reserve items
4. Shipping Service → create shipment

If step 3 fails after payment succeeded:

* you now have inconsistent state:

  * money deducted
  * no inventory

In a monolith, you’d use a DB transaction:

```sql
BEGIN;
...
ROLLBACK;
```

But in microservices:

* each service has its own DB
* distributed DB transactions (2PC) are painful and slow

So we use **Saga**.

---

# Basic Idea

A saga is:

> A sequence of local transactions + compensating actions.

Example:

| Step | Action            | Compensation      |
| ---- | ----------------- | ----------------- |
| 1    | Create Order      | Cancel Order      |
| 2    | Charge Payment    | Refund Payment    |
| 3    | Reserve Inventory | Release Inventory |
| 4    | Create Shipment   | Cancel Shipment   |

If step 3 fails:

* run compensation for step 2
* then compensation for step 1

---

# Two Main Saga Styles

## 1. Choreography Saga

Services communicate through events.

No central coordinator.

Example flow:

```text
Order Created Event
    ↓
Payment Service charges money
    ↓
Payment Success Event
    ↓
Inventory Service reserves stock
    ↓
Inventory Failed Event
    ↓
Payment Service refunds
    ↓
Order Service cancels order
```

Each service listens to events and reacts.

---

## 2. Orchestration Saga

There is a central **Saga Orchestrator**.

It controls the workflow.

```text
Orchestrator:
  Step 1 -> Order Service
  Step 2 -> Payment Service
  Step 3 -> Inventory Service
```

If inventory fails:

```text
Orchestrator:
  Refund Payment
  Cancel Order
```

Much easier to debug and understand.

---

# Real Example

Imagine booking a trip:

1. Book flight
2. Book hotel
3. Book cab

If hotel booking fails:

* cancel flight
* cancel cab

That whole workflow is a saga.

---

# Important Concepts

## Compensation != Rollback

Rollback in databases:

* automatic
* immediate
* exact previous state restoration

Compensation in sagas:

* business-level undo
* may not perfectly restore state

Example:

* refund may take 2 days
* email already sent
* external side effects happened

---

# Saga vs 2PC (Two-Phase Commit)

| Saga                       | 2PC                      |
| -------------------------- | ------------------------ |
| Eventually consistent      | Strong consistency       |
| Faster                     | Slower                   |
| Scales better              | Hard to scale            |
| Preferred in microservices | Rare nowadays            |
| Uses compensation          | Uses distributed locking |

Modern systems usually prefer saga.

---

# Problems in Sagas

## 1. Retry Issues

What if:

* payment succeeded
* response got lost

Need:

* idempotency
* retries
* deduplication

---

## 2. Ordering Problems

Events may arrive out of order.

Need:

* event versioning
* sequencing

---

## 3. Partial Failures

Compensation itself may fail.

Example:

* refund service down

Now you need:

* retry queues
* dead letter queues
* manual intervention sometimes

---

# Where Sagas Are Used

Very common in:

* e-commerce
* fintech
* logistics
* food delivery
* ride booking
* travel booking

Basically:

> multi-step workflows across services

---

# Technologies Commonly Used

* Apache Kafka
* RabbitMQ
* Temporal
* Camunda
* AWS Step Functions

---

# Simple Mental Model

Think of Saga as:

> "A distributed try-catch with undo steps."

```text
try:
   create_order()
   charge_payment()
   reserve_inventory()
except:
   refund_payment()
   cancel_order()
```

But spread across multiple services and async systems.

