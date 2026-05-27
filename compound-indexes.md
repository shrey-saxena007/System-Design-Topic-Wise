A **compound index** is a database index created on **multiple fields/columns together** instead of just one.

For example, in MongoDB:

```js
db.users.createIndex({ country: 1, age: -1 })
```

This creates an index on:

1. `country` (ascending)
2. `age` (descending)

---

# Why do we need it?

Suppose you frequently run queries like:

```js
db.users.find({
  country: "India",
  age: { $gt: 25 }
})
```

Without an index:

* MongoDB scans the whole collection (slow).

With a compound index:

* MongoDB can quickly jump to:

  * all `"India"` records
  * then efficiently filter by `age`.

---

# Real-world analogy

Think of a phone book sorted by:

1. City
2. Then Name

If you want:

> "People in Bangalore whose name starts with S"

You can find them very fast because the book is already sorted in that order.

That’s exactly what a compound index does.

---

# Where compound indexes are used

## 1. Filtering on multiple fields

Very common case.

Example:

```js
find({
  status: "ACTIVE",
  type: "PREMIUM"
})
```

Index:

```js
{ status: 1, type: 1 }
```

---

## 2. Sorting + filtering together

Example:

```js
find({ userId: 10 })
.sort({ createdAt: -1 })
```

Index:

```js
{ userId: 1, createdAt: -1 }
```

Very common in:

* feeds
* chats
* order history
* logs

---

## 3. Multi-tenant systems

Example:

```js
find({
  tenantId: "t1",
  orderId: 123
})
```

Index:

```js
{ tenantId: 1, orderId: 1 }
```

Used heavily in SaaS products.

---

## 4. Preventing duplicate combinations

You can make compound indexes unique.

Example:

```js
{ email: 1, companyId: 1 }
```

This means:

* same email allowed in different companies
* but not twice in same company

---

# Important concept: Prefix Rule

Very important in compound indexes.

Suppose index is:

```js
{ a: 1, b: 1, c: 1 }
```

This index can help queries on:

✅ `a`
✅ `a + b`
✅ `a + b + c`

But NOT efficiently on:

❌ only `b`
❌ only `c`

Because indexes work left-to-right.

---

# Example

Index:

```js
{ country: 1, city: 1 }
```

Good query:

```js
find({ country: "India" })
```

Also good:

```js
find({
  country: "India",
  city: "Bangalore"
})
```

Not efficient:

```js
find({ city: "Bangalore" })
```

because `country` comes first.

---

# When NOT to use compound indexes

Avoid creating too many because:

* indexes consume RAM
* writes become slower
* inserts/updates must update every index

Bad practice:

```js
{ a:1, b:1 }
{ a:1, c:1 }
{ a:1, d:1 }
{ a:1, e:1 }
```

Too many indexes can hurt performance.

---

# Typical backend examples

## E-commerce

```js
{ category: 1, price: 1 }
```

Filter by category and sort by price.

---

## Logistics / trip systems

Since you work with trips and routes:

```js
{ source: 1, destination: 1, vehicleType: 1 }
```

Useful for:

* historical trip lookup
* pricing lookup
* route analytics

Very similar to the fallback pricing problem you mentioned earlier.

---

## Chat apps

```js
{ conversationId: 1, createdAt: -1 }
```

Fetch latest messages fast.

---

# Single vs Compound index

| Type           | Example                      | Use                 |
| -------------- | ---------------------------- | ------------------- |
| Single index   | `{ email: 1 }`               | Search by one field |
| Compound index | `{ status:1, createdAt:-1 }` | Multi-field queries |

---

# Rule of thumb

Create compound indexes based on:

1. Most frequent queries
2. Filter fields
3. Sort fields
4. Join/lookups

Indexes should match how your application queries data.

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
---------------------------------------------


Question-: can we create indexes on primary key? and is it even useful
Answer-: Whenever we create a primary key, most database themselves create its indexes by their own, so we don't need to do that manually. Yes, they are useful, there are many cases where we need to find the orders by userId, address by userId, this userId is primary key, so it needs to be indexed

Question-: Indexes make searching faster, why don't we create indexes on every attribute?
Answr-: if we create index on 20 attributes, then insertion of one doc, will insert the doc in 20 Btress, which will take long time. So write operations take lot of time if we increase indexes
