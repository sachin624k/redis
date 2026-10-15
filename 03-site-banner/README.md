# 03 — Site Banner Using Redis

## 1. Why Use Redis for a Site Banner?

A site banner or **Hello Bar** displays messages such as _"Welcome to our website!"_ or _"Festive Sale — 50% Off!"_.

- Serving dynamic banner data directly from the database on every request adds unnecessary database load.
- Redis can store the banner message in memory for fast access.
- The backend reads and updates the banner through Redis instead of querying the main database for every request.
- Static banners can also be served directly from the frontend. Redis is useful when we want to change banner content dynamically without redeploying the frontend.

**Flow:**

```text
Client → Backend → Redis → Banner Response
```

**Note:** Keep the dependency versions installed by npm if they differ from the example.

## 2. Important Redis Code

### A. Define the Redis Key

```js
const BANNER_KEY = "app:banner";
```

- A Redis key identifies where a value is stored.
- `app:banner` is the key used to store our banner message.
- `app` represents the application, and `banner` describes the data.
- The `:` separator is a common naming convention, not a special Redis operator.

**Why use a constant?**

Defining the key once avoids repeating string literals and reduces spelling mistakes. In larger projects, Redis keys are often organized in a separate constants or enums file.

### B. Set the Banner — POST

```js
app.post("/banner", async (req, res) => {
  await redis.set(BANNER_KEY, req.body.message || "Welcome to our website!");

  res.json({ success: true });
});
```

- `POST /banner` creates or updates the banner message.
- `req.body.message` reads the message from the request body.
- `redis.set(key, value)` stores the message in Redis.
- If no message is provided or the value is empty, the default message is used.
- `await` waits for Redis to complete the operation.

Example Redis command:

```redis
SET app:banner "Welcome to our website!"
```

**Remember:** Redis `SET` creates the key if it doesn't exist and overwrites its previous value if it does.

### C. Get the Banner — GET

```js
app.get("/banner", async (req, res) => {
  const message = await redis.get(BANNER_KEY);
  res.json({ message });
});
```

- `GET /banner` retrieves the banner message.
- `redis.get(key)` reads the value stored against the key.
- If the key doesn't exist, Redis returns `null`.
- The backend sends the result as a JSON response.

Example Redis command:

```redis
GET app:banner
```

Example response:

```json
{
  "message": "Welcome to our website!"
}
```

### D. Delete the Banner — DELETE

```js
app.delete("/banner", async (req, res) => {
  await redis.del(BANNER_KEY);
  res.json({ success: true });
});
```

- `DELETE /banner` removes the banner key from Redis.
- `redis.del(key)` deletes the key and its stored value.
- If the key doesn't exist, Redis simply reports that nothing was deleted.
- The API returns `{ "success": true }` after the command completes.

Example Redis command:

```redis
DEL app:banner
```

**Important:** Deleting the key means the next GET request returns `null`; it does not automatically restore the default message.

### E. Check Whether the Banner Exists

```js
app.get("/banner/exists", async (req, res) => {
  const exists = await redis.exists(BANNER_KEY);
  res.json({ exists: Boolean(exists) });
});
```

- `redis.exists(key)` checks whether the key exists.
- Redis returns `1` if the key exists and `0` if it doesn't.
- `Boolean(exists)` converts that number into `true` or `false`.

Example Redis command:

```redis
EXISTS app:banner
```

Example response when the key exists:

```json
{
  "exists": true
}
```

Example response when the key doesn't exist:

```json
{
  "exists": false
}
```

## 3. Complete API Reference

| Method | Endpoint         | Purpose                         |
| ------ | ---------------- | ------------------------------- |
| POST   | `/banner`        | Create or update the banner     |
| GET    | `/banner`        | Retrieve the banner             |
| DELETE | `/banner`        | Delete the banner               |
| GET    | `/banner/exists` | Check whether the banner exists |

## 4. Test Using Postman

### Step 1 — Start Redis

From the root `Redis` directory:

```bash
docker compose up -d
```

### Step 2 — Start the Backend

Open a terminal in `03-site-banner`:

```bash
npm run dev
```

The server listens on port `3000`.

### Step 3 — Create or Update the Banner

- Method: `POST`
- URL: `http://localhost:3000/banner`
- Body → raw → JSON

```json
{
  "message": "Festive Sale! Get 50% Off."
}
```

Expected response:

```json
{
  "success": true
}
```

### Step 4 — Retrieve the Banner

- Method: `GET`
- URL: `http://localhost:3000/banner`

Expected response:

```json
{
  "message": "Festive Sale! Get 50% Off."
}
```

### Step 5 — Check Whether It Exists

- Method: `GET`
- URL: `http://localhost:3000/banner/exists`

Expected response:

```json
{
  "exists": true
}
```

### Step 6 — Delete the Banner

- Method: `DELETE`
- URL: `http://localhost:3000/banner`

Expected response:

```json
{
  "success": true
}
```

Now call `GET /banner/exists` again. The response should be:

```json
{
  "exists": false
}
```

## 5. Important Concepts to Remember

- **Redis key:** A unique identifier used to access a stored value.
- **`SET`:** Stores or updates a value.
- **`GET`:** Retrieves a value.
- **`DEL`:** Deletes a key.
- **`EXISTS`:** Checks whether a key exists.
- **`ioredis`:** Node.js client used to communicate with Redis.
- **`BANNER_KEY`:** A constant that avoids repeating the Redis key string.
- **`await`:** Waits for an asynchronous Redis operation to finish.
- **Dynamic banner:** Content can be updated through an API without changing frontend code.

## 6. Limitations and Production Notes

- Redis is being used as the primary storage for this demonstration, not as a cache backed by MongoDB.
- Redis data can be lost if persistence is not configured appropriately. This project enables AOF persistence in Docker, but durability depends on the configuration and failure scenario.
- The example has no authentication or authorization. In a real application, restrict who can create, update, or delete banners.
- Add input validation and error handling before production use.
- If a banner should expire automatically, Redis supports TTL through commands such as `SET key value EX 3600`.

**Core idea:** Store the banner message in Redis and access it through backend API endpoints, avoiding a database query for every banner request.
