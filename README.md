# HackTickets — Decentralized Ticketing System

A Web2 + Web3 ticketing platform that combines a familiar mobile app experience with on-chain ticket verification. No crypto wallet required for users.

## Repositories

| Repo | Description |
|------|-------------|
| [`htContract`](../htContract) | Solidity smart contract (Hardhat) — ticket registry on-chain |
| [`htBe`](../htBe) | Express.js backend API + admin web panel (EJS) |
| [`htApp`](../htApp) | Ionic/React user mobile app |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  User Mobile App (Ionic/React — htApp)                      │
│  Browse events · Login with phone+OTP · Book ticket · QR   │
└────────────────────────┬────────────────────────────────────┘
                         │ REST API (/api/*)
┌────────────────────────▼────────────────────────────────────┐
│  Backend API + Admin Panel (Express.js — htBe)              │
│  Auth · Events · Tickets · Entry verification               │
│  Admin Panel: /admin (EJS views, QR scanner)                │
└────────────────────────┬────────────────────────────────────┘
                         │ ethers.js
┌────────────────────────▼────────────────────────────────────┐
│  Smart Contract (Solidity — htContract)                     │
│  DecentralizedTicketRegistry on Hardhat local / Base Sepolia│
└─────────────────────────────────────────────────────────────┘
```

## Quick Start (Local Development)

### 1. Install all dependencies

```bash
cd htContract && npm install
cd ../htBe    && npm install
cd ../htApp   && npm install
```

### 2. Terminal 1 — Start local blockchain

```bash
cd htContract
npm run node
# Hardhat node running on http://127.0.0.1:8545
```

### 3. Terminal 2 — Deploy the contract

```bash
cd htContract
npm run deploy
# Writes contract address + ABI to htBe/deployment.json
```

### 4. Terminal 3 — Start backend

```bash
cd htBe
cp .env.example .env   # fill in your values
npm run dev
# API server on http://localhost:3000
```

### 5. Terminal 4 — Start user app

```bash
cd htApp
npm run dev
# Ionic/Vite dev server on http://localhost:5173
```

## Access Points

| URL | Description |
|-----|-------------|
| `http://localhost:5173` | User mobile app (Ionic) |
| `http://localhost:3000/admin` | Admin dashboard |
| `http://localhost:3000/admin/login` | Admin login |
| `http://localhost:3000/admin/scanner` | QR entry scanner |
| `http://localhost:3000/api/health` | API health check |

## API Endpoints

### Auth
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/send-otp` | — | Send OTP to phone number |
| POST | `/api/auth/verify-otp` | — | Verify OTP, returns JWT |

### Events
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/events` | — | Get all events |
| GET | `/api/events/:id` | — | Get event by ID |
| POST | `/api/events` | Admin JWT | Create event (on-chain) |
| PATCH | `/api/events/:id/status` | Admin JWT | Activate / deactivate event |

### Tickets
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/tickets/book` | User JWT | Book a ticket (mints on-chain) |
| GET | `/api/tickets/my-bookings` | User JWT | Get caller's tickets |
| GET | `/api/tickets/:id` | User JWT | Get ticket by ID |

### Entry Verification
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/verifyme/:ticketId/:userToken` | Admin JWT | Look up ticket info from QR |
| POST | `/api/entry/confirm` | Admin JWT | Confirm entry (marks ticket used on-chain) |

### Admin
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/health` | — | System health + chain info |
| GET | `/api/admin/stats` | Admin JWT | Dashboard stats |

## Environment Variables

See `htBe/.env.example`:

```env
PORT=3000
RPC_URL=http://127.0.0.1:8545
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
SALT=your_secret_salt_here_change_this_in_production
```

> ⚠️ **Production**: Replace `PRIVATE_KEY` and `SALT` with secure values. Update `RPC_URL` to Base Sepolia or your target network.

## How It Works

1. **Admin creates an event** on-chain via the admin dashboard — name, location, datetime, price (Wei), capacity.
2. **User browses events** in the app, taps "Book Now", completes the demo payment form.
3. **Backend mints a ticket** on-chain — stores a hash of `(phone + eventId + salt)` for privacy.
4. **App shows a QR code** encoding `{backendUrl}/verifyme/{ticketId}/{userJWT}`.
5. **Admin scans QR** (or pastes the URL) in the scanner page — backend returns ticket info.
6. **Admin confirms entry** — backend calls the contract to mark the ticket as used, preventing re-entry.

## Security Notes

- Phone numbers are **never stored in plain text** — only a salted hash goes on-chain.
- Payment references are hashed to prevent replay.
- JWTs expire after 24 h.
- Only the contract owner (backend wallet) can mint tickets and mark them used.
- OTPs are logged to the console in dev mode (no SMS dependency needed for demo).

## Testing with Postman

Import `Ticketing_System_API.postman_collection.json` into Postman. The collection covers the full flow: health → create event → send OTP → verify OTP → book ticket → verify entry → confirm entry.