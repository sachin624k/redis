# 05 — Storing User Profile: JSON vs Hash

## 1. What Are We Learning?

Redis can store user profiles in two common ways:

1. **JSON String:** Stores the entire object as a single serialized string.
2. **Redis Hash:** Stores the object as individual field-value pairs.

Example user profile:

```json
{
  "name": "Sachin",
  "email": "sachin@example.com",
  "age": 20
}
```

## 2. Approach 1: Store as JSON

### Save User Profile

```js
app.post("/user/:id/json", async (req, res) => {
  await redis.set(`user:${req.params.id}:json`, JSON.stringify(req.body));

  res.json({ savedAs: "json" });
});
```

**Important parts:**

- `req.params.id` → Gets the user ID from the URL.
- `req.body` → Contains the profile sent by the client.
- `JSON.stringify()` → Converts the JavaScript object into a JSON string.
- `redis.set()` → Stores the string in Redis.
- `res.json()` → Sends a response to the client.

Example Redis key: `user:101:json`

### Get User Profile

```js
app.get("/user/:id/json", async (req, res) => {
  const raw = await redis.get(`user:${req.params.id}:json`);

  res.json({ user: raw ? JSON.parse(raw) : null });
});
```

**Important parts:**

- `redis.get()` → Retrieves the stored string.
- `JSON.parse(raw)` → Converts the JSON string back into a JavaScript object.
- `raw ? JSON.parse(raw) : null` → Returns the parsed object if found; otherwise, returns `null`.

**Remember:**

- `JSON.stringify()` → Object to string.
- `JSON.parse()` → String to object.

## 3. Approach 2: Store as Redis Hash

A Redis Hash stores data as individual field-value pairs under one key.

Example:

```text
Key: user:101:hash

name  → Sachin
email → sachin@example.com
age   → 20
```

### Save User Profile

```js
app.post("/user/:id/hash", async (req, res) => {
  const fields = Object.fromEntries(
    Object.entries(req.body).map(([key, value]) => [key, String(value)]),
  );

  await redis.hset(`user:${req.params.id}:hash`, fields);

  res.json({ savedAs: "hash" });
});
```

**Important parts:**

- `redis.hset()` → Creates or updates fields in a Redis Hash.
- `Object.entries()` → Converts the object's properties into key-value pairs.
- `String(value)` → Converts simple field values into strings.
- `Object.fromEntries()` → Builds an object suitable for passing to `HSET`.

This example supports simple values such as strings and numbers. Nested objects need a separate serialization strategy.

### Get User Profile

```js
app.get("/user/:id/hash", async (req, res) => {
  const user = await redis.hgetall(`user:${req.params.id}:hash`);

  res.json({ user });
});
```

**Important parts:**

- `redis.hgetall()` → Retrieves all fields and values from the Hash.
- `user` → Contains the retrieved fields as an object.
- `res.json()` → Returns the object to the client.

**Remember:** Redis Hash values are strings, so a number such as `20` is returned as `"20"`.

## 4. The Main Learning: What Changes Behind the Scenes?

When you test both approaches using Postman, the API responses may look almost identical.

For example:

```json
{
  "user": {
    "name": "Sachin",
    "email": "sachin@example.com",
    "age": "20"
  }
}
```

The main difference is **the internal storage architecture and how we manipulate the data**.

### JSON Storage

```text
User Object
    ↓
JSON.stringify()
    ↓
Redis SET
    ↓
Entire object stored as one string
    ↓
GET → JSON.parse()
```

If you want to update one field, you generally retrieve the entire JSON string, parse it, modify the object, and store it again.

### Hash Storage

```text
User Object
    ↓
Redis HSET
    ↓
Fields stored individually
    ↓
HGETALL → Entire object
```

You can update a single field directly without rewriting the entire profile.

For example:

```js
await redis.hset("user:101:hash", "name", "Sachin Kumar");
```

Only the `name` field changes; the other fields remain untouched.

**Key takeaway:** Postman shows us the API's input and output. The real architectural difference is how Redis stores, retrieves, and updates the data internally.

## 5. Important Redis Hash Methods

Redis provides several commands to work with individual fields.

| Method    | Purpose                      | Example                             |
| --------- | ---------------------------- | ----------------------------------- |
| `HSET`    | Create or update fields      | `redis.hset(key, "name", "Sachin")` |
| `HGET`    | Retrieve one field           | `redis.hget(key, "name")`           |
| `HGETALL` | Retrieve all fields          | `redis.hgetall(key)`                |
| `HDEL`    | Delete one or more fields    | `redis.hdel(key, "age")`            |
| `HEXISTS` | Check whether a field exists | `redis.hexists(key, "email")`       |
| `HKEYS`   | Get all field names          | `redis.hkeys(key)`                  |
| `HVALS`   | Get all field values         | `redis.hvals(key)`                  |
| `HLEN`    | Count fields in a Hash       | `redis.hlen(key)`                   |
| `HINCRBY` | Increment a numeric field    | `redis.hincrby(key, "age", 1)`      |

### A. HGET — Get One Field

```js
const name = await redis.hget("user:101:hash", "name");
```

Returns the value of the `name` field.

### B. HDEL — Delete a Field

```js
await redis.hdel("user:101:hash", "age");
```

Deletes only the `age` field. Other fields remain unchanged.

### C. HEXISTS — Check a Field

```js
const exists = await redis.hexists("user:101:hash", "email");
```

Returns:

- `1` → Field exists.
- `0` → Field does not exist.

### D. HKEYS — Get All Field Names

```js
const fields = await redis.hkeys("user:101:hash");
```

Example result:

```json
["name", "email", "age"]
```

### E. HLEN — Count Fields

```js
const count = await redis.hlen("user:101:hash");
```

If the Hash contains three fields, the result is `3`.

### F. HINCRBY — Increment a Numeric Field

```js
await redis.hincrby("user:101:hash", "age", 1);
```

Increments the `age` field by one, provided its value is a valid integer.

**Important:** These methods operate on fields within a Redis Hash. They are not interchangeable with ordinary string commands such as `GET` and `SET`.

## 6. JSON vs Hash — Quick Comparison

| Feature               | JSON String                        | Redis Hash                 |
| --------------------- | ---------------------------------- | -------------------------- |
| Save                  | `SET`                              | `HSET`                     |
| Retrieve everything   | `GET` + `JSON.parse()`             | `HGETALL`                  |
| Retrieve one field    | Parse the entire JSON string       | `HGET`                     |
| Update one field      | Modify and rewrite the JSON string | `HSET`                     |
| Delete one field      | Modify and rewrite the JSON string | `HDEL`                     |
| Check field existence | Inspect the parsed object          | `HEXISTS`                  |
| Storage structure     | One serialized string              | Multiple field-value pairs |

Both approaches can be useful. The right choice depends on how you access and update your data.

## 7. Test Using Postman

### A. Save as JSON

**POST** `http://localhost:3000/user/101/json`

Body → raw → JSON:

```json
{
  "name": "Sachin",
  "email": "sachin@example.com",
  "age": 20
}
```

### B. Retrieve JSON

**GET** `http://localhost:3000/user/101/json`

### C. Save as Hash

**POST** `http://localhost:3000/user/101/hash`

Send the same JSON body.

### D. Retrieve Hash

**GET** `http://localhost:3000/user/101/hash`

### What should you observe?

Both GET endpoints return the user's profile, but the internal Redis commands and data structures differ.

To understand the difference, focus on the code and the Redis commands—not just the Postman response.

## 8. Quick Revision

```text
JSON:
SET       → Store the complete JSON string
GET       → Retrieve the string
STRINGIFY → Object to string
PARSE     → String to object

HASH:
HSET      → Create or update fields
HGET      → Retrieve one field
HGETALL   → Retrieve all fields
HDEL      → Delete a field
HEXISTS   → Check whether a field exists
HKEYS     → Retrieve field names
HVALS     → Retrieve field values
HLEN      → Count fields
HINCRBY   → Increment a numeric field
```

**Final takeaway:** JSON stores the entire profile as one string, whereas Redis Hash stores individual fields. Both can return similar API responses, but Hash makes individual field operations more convenient.

**Note:** In a real application, Redis is often used as a cache alongside MongoDB or PostgreSQL. Changes to cached data must be coordinated with the main database to avoid stale or inconsistent data.
