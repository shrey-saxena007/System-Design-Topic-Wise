Retries are one of the most common reliability mechanisms in distributed systems.

The basic idea is:

> If a request fails temporarily, try again instead of immediately failing.

This sounds simple, but retry design becomes extremely important at scale because retries themselves can destroy systems if done badly.

---

# Why retries are needed

In distributed systems, failures are normal:

* network packets drop
* services restart
* databases become slow
* pods get rescheduled
* TCP connections reset
* timeouts happen
* load balancers route to dead instances

A request failing once does **not** always mean the operation is impossible.

So clients retry.

---

# Simple retry flow

Suppose:

Service A → calls → Service B

```text
A ---- request ----> B
A <--- timeout -----
```

A didn't get response.

Now A retries:

```text
A ---- retry request ----> B
A <---- success ----------
```

Maybe:

* B was temporarily overloaded
* network hiccup happened
* connection pool was exhausted for 100ms

Retry recovered automatically.

---

# Important problem: duplicate execution

This is THE biggest retry problem.

Imagine payment service:

```text
Charge ₹100
```

Client sends request.

Server processes payment successfully.

BUT response gets lost.

Client thinks:

> "Request failed, retry."

Now payment may happen twice.

This is why retries are dangerous for:

* payments
* order creation
* inventory deduction
* ticket booking

---

# Idempotency

To safely retry operations, systems use:

## Idempotent operations

Meaning:

> Doing the same operation multiple times produces same final result.

Example:

```http
PUT /user/1/name = "shrey"
```

Running 10 times:

* final state still same

Good for retries.

---

## Non-idempotent operations

Example:

```text
Add ₹100 to wallet
```

Retrying:

* balance increases again

Bad.

---

# Idempotency keys

Very common pattern.

Client generates unique request ID:

```text
request-id: abc123
```

Server stores processed IDs.

If same retry comes again:

```text
"Oh, already processed."
Return previous response.
```

This makes retries safe.

Used heavily in:

* payment systems
* Stripe APIs
* order systems
* booking systems

---

# Retry policies

Retries are usually configurable.

---

## 1. Immediate retry

```text
try again instantly
```

Good for:

* tiny transient network failures

Bad:

* can overload already struggling service

---

## 2. Fixed delay retry

```text
Retry every 2 seconds
```

Simple but not ideal.

---

## 3. Exponential backoff

Most common.

Example:

```text
1st retry -> 1 sec
2nd retry -> 2 sec
3rd retry -> 4 sec
4th retry -> 8 sec
```

Why?

Because if service is overloaded:

* aggressive retries make it worse

Backoff gives service time to recover.

---

# Jitter

Very important in real systems.

Without jitter:

Imagine 10,000 clients retrying exactly after 2 seconds.

You create another traffic spike.

So systems add randomness:

```text
retry after 2.3 sec
retry after 1.7 sec
retry after 2.8 sec
```

This spreads load.

Called:

* retry jitter
* randomized backoff

---

# Retry storms

Classic distributed systems disaster.

Scenario:

* DB slows down
* requests timeout
* all services retry
* retries increase load
* DB slows even more
* more retries happen

System collapses.

This is called:

* retry storm
* thundering herd

---

# How systems prevent retry storms

## Circuit breakers

If service failing continuously:

```text
STOP sending requests temporarily
```

Like electrical circuit breaker.

Example:

* after 50 failures
* open circuit for 30 sec

Used in:

* Netflix Hystrix (famous)
* resilience4j

---

## Retry limits

Never retry infinitely.

Example:

```text
max retries = 3
```

---

## Timeouts

Always combine retries with timeout.

Otherwise request may hang forever.

---

# Which errors should be retried?

Very important.

---

## Retryable errors

Usually:

* network timeout
* connection reset
* 503 Service Unavailable
* temporary DB failover
* leader election happening

---

## Non-retryable errors

Usually:

* 400 Bad Request
* validation failed
* authentication failed
* syntax errors

Retrying these is useless.

---

# Retries in message queues

Very common.

Suppose consumer fails processing message.

Queue retries later.

Example systems:

* Apache Kafka
* RabbitMQ
* Amazon SQS

Flow:

```text
consume message
↓
processing fails
↓
requeue message
↓
retry later
```

---

# Dead Letter Queue (DLQ)

What if message always fails?

Example:

* corrupted payload
* bad data

Infinite retries are bad.

So after N retries:

```text
move message to DLQ
```

Dead Letter Queue stores poison messages for inspection.

---

# Retries + microservices

Suppose:

```text
API Gateway
   ↓
Order Service
   ↓
Payment Service
   ↓
Bank API
```

Where should retries happen?

This matters a LOT.

Because layered retries multiply.

Example:

```text
3 retries at each layer
```

Total:

```text
3 × 3 × 3 = 27 requests
```

This can kill downstream systems.

So retries are usually centralized carefully.

---

# Common real-world strategy

Typical production strategy:

```text
- timeout: 2 sec
- retries: 2 or 3
- exponential backoff
- jitter
- idempotency keys
- circuit breaker
```

---

# One subtle thing: "at least once"

Retries often mean:

> Operation may execute multiple times.

Distributed systems usually choose between:

| Guarantee     | Meaning                      |
| ------------- | ---------------------------- |
| At most once  | maybe lost, never duplicated |
| At least once | never lost, may duplicate    |
| Exactly once  | hardest and expensive        |

Retries naturally push systems toward:

* at least once delivery

Which is why idempotency becomes critical.

---

# Easy mental model

Retries are basically:

> "Assume failures are temporary, but protect system from duplicate work and overload."

That balance is the heart of retry design in distributed systems.

---
---
---
---
---
---
---
---
---
---
---
---
---
---
---
# Retries in payment using idempotency

Let’s walk through a realistic payment flow with idempotency keys step-by-step.

This is one of the most important distributed systems patterns in fintech.

We’ll model something similar to how Stripe or modern payment gateways behave internally.

---

# Problem we are solving

Suppose user clicks:

```text id="1gqjlwm"
Pay ₹1000
```

Client sends request.

But:

* network timeout happens
* response lost
* mobile app retries
* user double-clicks button

Without protection:

```text id="q7ujd0"
money deducted twice
```

Idempotency keys prevent this.

---

# High-level idea

Every logical payment attempt gets a unique key.

Example:

```text id="g91cll"
idempotency_key = "pay_8f92ab"
```

Server guarantees:

> Same key → same logical operation → never execute twice.

---

# Full payment flow

---

# Step 1: User clicks Pay

Frontend/app generates unique key.

Example:

```text id="r3g2pn"
pay_8f92ab
```

Usually:

* UUID
* random token
* globally unique

---

# Step 2: Client sends payment request

```http id="f61v09"
POST /payments

Headers:
Idempotency-Key: pay_8f92ab

Body:
{
  "amount": 1000,
  "currency": "INR",
  "user_id": 42
}
```

---

# Step 3: Payment service receives request

Now server checks:

```text id="eg2mgh"
Have I seen this idempotency key before?
```

Typically database table:

| idempotency_key | status | response |
| --------------- | ------ | -------- |
| pay_8f92ab      | ?      | ?        |

---

# CASE 1 — Key not found (first request)

Server inserts initial record:

| idempotency_key | status     |
| --------------- | ---------- |
| pay_8f92ab      | PROCESSING |

This is IMPORTANT.

Why?

Because concurrent retries may arrive immediately.

---

# Step 4: Server starts actual payment

Flow may be:

```text id="8m20s9"
Payment Service
    ↓
Bank API
    ↓
Card Network
    ↓
Bank
```

Money gets charged.

---

# Step 5: Payment succeeds

Server stores final result:

| idempotency_key | status  | response       |
| --------------- | ------- | -------------- |
| pay_8f92ab      | SUCCESS | payment_id=123 |

Now server returns:

```json id="nkl0of"
{
  "payment_id": 123,
  "status": "SUCCESS"
}
```

Done.

---

# Now comes the important part

Suppose response never reaches client.

---

# Step 6: Client retries

Because timeout happened.

Client sends SAME key:

```http id="k55jlwm"
POST /payments

Idempotency-Key: pay_8f92ab
```

---

# Step 7: Server checks key again

Now DB contains:

| idempotency_key | status  | response       |
| --------------- | ------- | -------------- |
| pay_8f92ab      | SUCCESS | payment_id=123 |

So server DOES NOT charge again.

Instead:

```text id="hvv3kc"
Return previously stored response
```

Client gets:

```json id="2qj7r7"
{
  "payment_id": 123,
  "status": "SUCCESS"
}
```

No duplicate deduction.

---

# Core idea

The key represents:

```text id="3o3m6z"
logical operation
NOT
individual HTTP request
```

Many retries.
One actual effect.

---

# Important concurrency problem

Suppose retries happen VERY quickly.

Example:

```text id="78v5k8"
Request A arrives
Request B arrives 5ms later
```

Both check DB simultaneously.

Without proper locking:

```text id="ng0uwl"
both think key absent
both charge money
```

Disaster.

---

# How systems solve this

Usually with:

* unique DB constraint
* transactions
* distributed locks

---

# Typical DB design

Table:

```sql id="7mbm4l"
CREATE TABLE idempotency (
    idempotency_key TEXT PRIMARY KEY,
    status TEXT,
    response JSONB
);
```

Now duplicate insert fails automatically.

---

# Safe processing flow

Pseudo code:

```python id="0bqzye"
try:
    INSERT idempotency_key

except duplicate_key:
    return existing_response
```

This is atomic.

Very important.

---

# Another tricky case:

# Request crashes mid-processing

Suppose:

```text id="l4g1th"
PROCESSING
```

was stored.

Then service crashes before completion.

Now retry arrives.

Question:

```text id="0uhjlwm"
Should we retry?
Or payment already happened?
```

This becomes complicated.

---

# Real systems use state machines

Statuses like:

| Status     | Meaning         |
| ---------- | --------------- |
| PROCESSING | in progress     |
| SUCCESS    | completed       |
| FAILED     | safe failure    |
| UNKNOWN    | uncertain state |

---

# External payment gateway problem

Hardest scenario:

```text id="93t2y0"
Our server → Bank API
```

Suppose:

* bank charged card
* our server crashed before saving SUCCESS

Now retry comes.

We genuinely don't know:

* payment happened?
* or not?

---

# How real gateways solve this

Usually by:

* querying gateway using transaction ID
* reconciliation jobs
* webhook callbacks
* ledger systems

Example:

```text id="wjlwm2"
"Check transaction status"
```

before retrying actual charge.

---

# Idempotency key expiration

Systems don't store keys forever.

Typical expiry:

* 24 hours
* 48 hours
* 7 days

After expiry:

* same key may behave as new request

---

# Important distinction

Idempotency keys do NOT magically prevent duplicates everywhere.

They only work if:

* all retries use same key
* server stores state correctly
* downstream systems cooperate

---

# Real architecture

A realistic flow:

```text id="lrfd9u"
Mobile App
   ↓
API Gateway
   ↓
Payment Service
   ↓
Idempotency Store (Redis/DB)
   ↓
Bank Gateway
```

Sometimes Redis used for:

* fast lookup
* distributed locks

DB used for:

* durable storage

---

# Why payment systems love this approach

Because it handles:

* retries
* double clicks
* network failures
* mobile reconnects
* client crashes
* gateway timeouts

without duplicate charges.

---

# Very important insight

Idempotency does NOT mean:

```text id="5r6ct0"
request executed once
```

It means:

```text id="1k7vpl"
side effects happen once
```

The server may receive:

* 2 requests
* 10 retries
* duplicated packets

But:

* only one charge occurs

That’s the real goal.

