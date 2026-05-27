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

