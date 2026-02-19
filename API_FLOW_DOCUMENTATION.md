# Decentralized Ticketing System - API Flow Documentation

## Authentication & Security

The system uses **JWT tokens** with **OTP verification** for secure authentication.  
Phone numbers are **10 digits only, no `+` prefix** (e.g. `9876543210`).  
Phone numbers are hashed on-chain and never exposed after login.

---

## API Flow Overview

### 1. Authentication Flow

#### Step 1: Send OTP
```http
POST /api/auth/send-otp
Content-Type: application/json

{
  "phone": "9876543210"
}
```

**Response:**
```json
{
  "success": true,
  "message": "OTP sent successfully",
  "otp": "123456"
}
```
> `otp` is only returned in `NODE_ENV=development`. In production, it is sent via SMS.

---

#### Step 2: Verify OTP & Login
```http
POST /api/auth/verify-otp
Content-Type: application/json

{
  "phone": "9876543210",
  "otp": "123456"
}
```

**Response:**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "phone": "9876543210",
    "isAdmin": false
  }
}
```
> Store this `token` — it is used as the Bearer token for all protected routes **and** embedded in the QR code.

---

#### Step 3: Refresh Token
```http
POST /api/auth/refresh
Authorization: Bearer <JWT_TOKEN>
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

### 2. Event Discovery (Public)

#### Get All Events
```http
GET /api/events
```

**Response:**
```json
[
  {
    "eventId": 0,
    "name": "Tech Conference 2026",
    "location": "Convention Center, NYC",
    "date": "1740787200",
    "price": "100",
    "capacity": "500",
    "ticketsSold": "50",
    "active": true,
    "availableTickets": "450"
  }
]
```

---

#### Get Event Details
```http
GET /api/events/:eventId
```

**Response:**
```json
{
  "eventId": "0",
  "name": "Tech Conference 2026",
  "location": "Convention Center, NYC",
  "date": "1740787200",
  "price": "100",
  "capacity": "500",
  "ticketsSold": "50",
  "active": true,
  "availableTickets": "450"
}
```

---

### 3. Ticket Booking (Protected)

```http
POST /api/tickets/book
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "eventId": "0",
  "paymentReference": "payment_unique_12345"
}
```

**Response:**
```json
{
  "success": true,
  "ticketId": "1",
  "eventId": "0",
  "transactionHash": "0x...",
  "message": "Ticket booked successfully"
}
```

> After booking, generate a QR code for the user that encodes the URL:  
> `http://<server>/verifyme/<ticketId>/<userJwtToken>`

---

### 4. My Bookings (Protected)

#### Get All Bookings (across all events)
```http
GET /api/tickets/all-bookings
Authorization: Bearer <JWT_TOKEN>
```

**Response:**
```json
[
  {
    "ticketId": "1",
    "eventId": "0",
    "eventName": "Tech Conference 2026",
    "eventLocation": "Convention Center, NYC",
    "eventDate": "1740787200",
    "eventPrice": "100",
    "used": false,
    "paymentId": "0x..."
  }
]
```

---

#### Get Bookings for a Specific Event
```http
GET /api/tickets/my-bookings?eventId=0
Authorization: Bearer <JWT_TOKEN>
```

**Response:**
```json
[
  {
    "ticketId": "1",
    "eventId": "0",
    "eventName": "Tech Conference 2026",
    "eventLocation": "Convention Center, NYC",
    "eventDate": "1740787200",
    "used": false,
    "paymentId": "0x..."
  }
]
```

---

### 5. Entry Verification Flow (QR-based, Admin Only)

This is the 2-step flow for verifying a user's ticket at the venue gate.

```
User QR code encodes:
  http://<server>/verifyme/<ticketId>/<userJwtToken>
```

---

#### Step 1 — Admin Scans QR → Get Ticket Details + Trigger OTP

```http
GET /verifyme/:ticketId/:userToken
Authorization: Bearer <ADMIN_JWT_TOKEN>
```

`userToken` is the user's own JWT (embedded in the QR code at booking time).  
Admin must be authenticated with their own Bearer token.

Backend:
- Decodes `userToken` to get user's phone
- Validates the ticket on-chain belongs to that user and is unused
- Generates a 6-digit OTP, stores it keyed by `ticketId`
- Logs/sends OTP to the user's phone

**Response:**
```json
{
  "ticketId": "1",
  "eventId": "0",
  "eventName": "Tech Conference 2026",
  "eventLocation": "Convention Center, NYC",
  "eventDate": "1740787200",
  "used": false,
  "phone": "9876543210",
  "otp": "482910"
}
```

> `otp` is only returned in `NODE_ENV=development`.  
> Admin sees `phone`, asks the user **"what's your phone number?"** to verify verbally, then asks for their OTP.

---

#### Step 2 — Admin Confirms Entry with Phone + OTP

```http
POST /api/entry/confirm
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "ticketId": "1",
  "phone": "9876543210",
  "otp": "482910"
}
```

Backend:
- Verifies phone matches ticket owner from the stored OTP session
- Verifies OTP matches and is not expired
- Derives `eventId` from the stored session (no need to pass it)
- Calls `markAsUsed(ticketId, phoneHash)` on the smart contract

**Response:**
```json
{
  "success": true,
  "message": "Entry granted – ticket marked as used",
  "ticketId": "1",
  "transactionHash": "0x..."
}
```

---

### 6. Event Management (Admin Only)

#### Create Event
```http
POST /api/admin/events
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "name": "New Event",
  "location": "Venue",
  "date": 1740787200,
  "price": 50,
  "capacity": 100
}
```

**Response:**
```json
{
  "success": true,
  "eventId": "1",
  "transactionHash": "0x..."
}
```

---

#### Set Event Status (Enable / Disable)
```http
PATCH /api/admin/events/:eventId/status
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "active": false
}
```

**Response:**
```json
{
  "success": true,
  "eventId": "0",
  "active": false,
  "transactionHash": "0x..."
}
```

---

#### Get Ticket Details
```http
GET /api/admin/tickets/:ticketId
Authorization: Bearer <ADMIN_JWT_TOKEN>
```

**Response:**
```json
{
  "ticketId": "1",
  "eventId": "0",
  "eventName": "Tech Conference 2026",
  "eventLocation": "Convention Center, NYC",
  "eventDate": "1740787200",
  "phoneHash": "0x...",
  "used": false,
  "paymentId": "0x..."
}
```

---

#### Mark Ticket as Used (Direct — No OTP)
```http
POST /api/admin/tickets/:ticketId/use
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json

{
  "userPhone": "9876543210",
  "eventId": "0"
}
```

**Response:**
```json
{
  "success": true,
  "ticketId": "1",
  "transactionHash": "0x...",
  "message": "Ticket marked as used"
}
```

> Use this for manual override. For the standard gate flow, use `/api/entry/initiate` + `/api/entry/confirm`.

---

### 7. Health Check

```http
GET /api/health
```

**Response:**
```json
{
  "status": "healthy",
  "network": "unknown",
  "chainId": "31337",
  "signerAddress": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "signerBalance": "9999.99"
}
```

---

## Complete User Journey

### Regular User

1. **Login** — Send OTP → Verify OTP → Get JWT token
2. **Browse** — Get all events (public)
3. **Book** — `POST /api/tickets/book` → receive `ticketId`
4. **QR Code** — App generates QR encoding `http://<server>/verifyme/<ticketId>/<jwtToken>`
5. **At Venue** — Show QR to admin scanner
6. **Entry** — Admin scans → gets ticket+phone details (OTP sent to user) → asks "what's your phone?" → asks for OTP → confirms → entry granted

### Admin

1. **Login** — Same OTP flow using admin phone from `.env`
2. **Manage Events** — Create events, enable/disable
3. **Gate Entry** — Scan QR (`GET /verifyme`) → get ticket+phone (OTP sent to user) → ask user for phone+OTP → confirm (`POST /api/entry/confirm`) → ticket marked used on-chain

---

## Security Features

| Feature | Details |
|---|---|
| JWT Authentication | All protected routes require a valid Bearer token |
| 10-digit phone only | No `+` prefix — enforced by `isValidPhone()` |
| Phone privacy | Phone is hashed (`keccak256(phone + eventId + salt)`) — never stored raw on-chain |
| Entry OTP | 2-step admin confirmation prevents unauthorized entry scans |
| Rate limiting | OTP: 5 req/15 min · Login: 10 req/15 min |
| Payment deduplication | `paymentId` is hashed and checked on-chain for uniqueness |
| OTP expiry | All OTPs expire after `OTP_EXPIRES_IN` ms (default 5 min) |

---

## Environment Variables

```env
# Server
PORT=3000

# Blockchain
RPC_URL=http://127.0.0.1:8545
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

# Security
SALT=dev_salt_change_in_production
JWT_SECRET=your_jwt_secret_key_change_in_production
JWT_EXPIRES_IN=24h

# OTP
OTP_EXPIRES_IN=300000
ADMIN_PHONE=1234567890
```

> `ADMIN_PHONE` is 10 digits — no `+` prefix.

---

## Error Handling

All endpoints return consistent error responses:

```json
{
  "error": "Error message description"
}
```

| Status | Meaning |
|---|---|
| `200` | Success |
| `400` | Bad Request (missing/invalid fields) |
| `401` | Unauthorized (missing or invalid token) |
| `403` | Forbidden (wrong user or insufficient permissions) |
| `404` | Not Found |
| `429` | Too Many Requests (rate limited) |
| `500` | Internal Server Error |

---

## Rate Limits

| Endpoint group | Limit |
|---|---|
| `POST /api/auth/send-otp` | 5 requests per 15 minutes per IP |
| `POST /api/auth/verify-otp` | 10 requests per 15 minutes per IP |

---

## Development vs Production

### Development (`NODE_ENV=development`)
- OTP is returned in API response and printed to console
- Uses default Hardhat local network accounts
- Entry OTP also returned in `/api/entry/initiate` response

### Production
- Integrate an SMS service for OTP delivery
- Use secure, randomly generated `PRIVATE_KEY`, `SALT`, `JWT_SECRET`
- Use Redis instead of in-memory Maps for OTP storage
- Enable HTTPS and strict CORS settings
- Remove OTP from all API responses