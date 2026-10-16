# 04 — Building OTP Verification with Redis

## 1. Why Use Redis for OTP Verification?

OTP (One-Time Password) verification is used to verify a user's phone number, email, or identity.

Redis is useful because it stores OTPs in memory and supports automatic expiration using TTL.

- Store OTPs temporarily in Redis.
- Verify the OTP when the user submits it.
- Automatically expire OTPs after a specified time.
- Delete OTPs after successful verification.
- Track remaining OTP lifetime using `TTL`.

**Basic flow:**

```text
User requests OTP
       ↓
Backend generates OTP
       ↓
Redis stores OTP with TTL
       ↓
User submits OTP
       ↓
Backend verifies OTP
       ↓
Success → Delete OTP
```

## 2. Dynamic Redis Keys

```js
function otpKey(phone) {
  return `otp:${phone}`;
}
```

### What does this do?

- `otpKey()` is a helper/utility function that generates Redis keys dynamically.
- `phone` is the phone number received from the request.
- Template literals (`${phone}`) insert the phone number into the string.

Example:

```js
otpKey("9876543210");
// "otp:9876543210"

otpKey("9123456780");
// "otp:9123456780"
```

Each phone number gets its own Redis key.

**Why use a helper function?**

Instead of repeating `"otp:" + phone` throughout the application, define the key format once and reuse it.

In larger projects, key-generation functions are commonly placed in a utility file, while fixed prefixes or key names may be stored in constants.

## 3. Generate and Store an OTP

```js
const otp = Math.floor(100000 + Math.random() * 900000).toString();

await redis.set(otpKey(phone), otp, "EX", 30);
```

### Important parts

- `Math.random()` generates a random decimal between 0 (inclusive) and 1 (exclusive).
- The formula generates a six-digit number.
- `.toString()` converts the number into a string.
- `redis.set()` stores the OTP in Redis.
- `"EX", 30` sets the key to expire after **30 seconds**.

Example Redis command:

```redis
SET otp:9876543210 "483921" EX 30
```

**Remember:** `EX` specifies the expiration time in seconds. Redis automatically removes the key when its TTL expires.

> Security note: `Math.random()` is suitable for this learning example, but production OTPs should be generated using Node.js's cryptographically secure `crypto` module. Never return the OTP in an API response in production; send it through the appropriate SMS or email provider.

## 4. Verify the OTP

```js
const savedOtp = await redis.get(otpKey(phone));

if (!savedOtp) {
  return res.status(400).json({
    message: "OTP expired or not found",
  });
}

if (savedOtp !== otp) {
  return res.status(400).json({
    message: "Invalid OTP",
  });
}

await redis.del(otpKey(phone));

res.json({ message: "OTP Verified" });
```

### How it works

1. **GET:** Retrieve the OTP stored for the phone number.
2. **Check existence:** If no OTP exists, it may have expired or never been created.
3. **Compare:** Check whether the submitted OTP matches the stored OTP.
4. **DEL:** Delete the OTP after successful verification.
5. **Respond:** Return a success message.

**Why delete the OTP?**

An OTP should be used only once. Deleting it after successful verification prevents the same code from being reused.

## 5. Check OTP Expiration Using TTL

```js
app.get("/otp/:phone/ttl", async (req, res) => {
  const ttl = await redis.ttl(otpKey(req.params.phone));
  res.json({ ttl });
});
```

### Important parts

- `:phone` is a route parameter.
- `req.params.phone` retrieves the phone number from the URL.
- `redis.ttl(key)` returns the remaining lifetime of that key in seconds.
- The response sends the remaining TTL as JSON.

Example request:

```http
GET http://localhost:3000/otp/9876543210/ttl
```

Example response:

```json
{
  "ttl": 18
}
```

Useful Redis TTL results:

- Positive number → remaining lifetime in seconds.
- `-1` → key exists but has no expiration.
- `-2` → key does not exist.

## 6. OTP Metadata — Advanced Concept

Instead of storing only an OTP string, you can store additional information to control verification attempts and lockouts.

Example metadata structure:

```js
{
  otp: "483921",
  attempts: 0,
  maxAttempts: 3,
  createdAt: "timestamp",
  lastAttemptAt: "timestamp",
  blockedUntil: "timestamp"
}
```

### What each field means

- `otp` → the generated verification code.
- `attempts` → number of incorrect verification attempts.
- `maxAttempts` → maximum allowed attempts, such as 3.
- `createdAt` → when the OTP was created.
- `lastAttemptAt` → when the last verification attempt occurred.
- `blockedUntil` → optional timestamp indicating when the user can try again.

**Important:** This metadata structure is a concept for an enhanced implementation; the current code stores only the OTP string.

To store an object in Redis, you can serialize it using `JSON.stringify()` and retrieve it using `JSON.parse()`. You must then manage attempts, blocking, expiration, and verification logic yourself.

## 7. API Reference

| Method | Endpoint          | Purpose                      |
| ------ | ----------------- | ---------------------------- |
| `POST` | `/otp`            | Generate and store an OTP    |
| `POST` | `/otp/verify`     | Verify the submitted OTP     |
| `GET`  | `/otp/:phone/ttl` | Check remaining OTP lifetime |

## 8. Test with Postman

Start Redis from the root `Redis` folder:

```bash
docker compose up -d
```

Start the backend from the `04-otp-verification` folder:

```bash
npm run dev
```

### A. Request an OTP

**POST** `http://localhost:3000/otp`

Body → raw → JSON:

```json
{
  "phone": "9876543210"
}
```

Example response:

```json
{
  "message": "OTP sent",
  "otp": "483921"
}
```

The example returns the OTP to simplify local testing. Production applications must send it through an SMS/email provider instead.

### B. Verify the OTP

**POST** `http://localhost:3000/otp/verify`

```json
{
  "phone": "9876543210",
  "otp": "483921"
}
```

Expected response:

```json
{
  "message": "OTP Verified"
}
```

### C. Check TTL

**GET** `http://localhost:3000/otp/9876543210/ttl`

Example response:

```json
{
  "ttl": 15
}
```

The value decreases as time passes.

### D. Test expiration

Wait until the OTP expires, then submit it again. You should receive:

```json
{
  "message": "OTP expired or not found"
}
```

## 9. Key Concepts to Remember

- **Dynamic key:** `otp:${phone}`
- **Generate OTP:** Create a six-digit code.
- **Store OTP:** `redis.set(key, value, "EX", 30)`
- **Retrieve OTP:** `redis.get(key)`
- **Delete OTP:** `redis.del(key)`
- **Check expiration:** `redis.ttl(key)`
- **One-time use:** Delete the OTP after successful verification.
- **Metadata:** Store attempts and timestamps when implementing stronger verification controls.

**Core idea:** Redis lets us store OTPs temporarily, automatically expire them, and remove them after successful verification. Additional metadata can support attempt limits and temporary lockouts.
