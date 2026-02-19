# Decentralized Ticketing System – MVP (Submission Version)

---

# 1. MVP Philosophy (Hackathon Version)

This MVP focuses on:

* Web2 user experience
* Blockchain transparency layer
* No wallet requirement
* No real SMS dependency
* No real payment dependency
* Fully demo-stable architecture

For demo reliability:

* OTP is generated and printed in server console (no SMS).
* Payment gateway is simulated (fake gateway).
* Admin panel runs from Express view engine.

---

# 2. Final Stack (MVP Version)

## Backend (Core System)

* Node.js
* Express.js
* ethers.js
* express-session or JWT
* crypto (OTP + hashing)
* EJS (or simple HTML) inside /views folder

## Frontend (User App – Served by Express)

* HTML
* CSS
* Vanilla JS
* Axios (optional)

## Admin Panel

* HTML (inside Express views folder)
* QR scanner (html5-qrcode)
* Calls backend APIs

## Blockchain

* Base Sepolia testnet
* Contract deployed via Hardhat

---

# 3. Smart Contract (Final MVP Version)

Contract: DecentralizedTicketRegistry

## Structs

EventData:

* name
* location
* date
* price
* capacity
* ticketsSold
* active

Ticket:

* eventId
* phoneHash
* used
* paymentId (bytes32)

## Storage

* mapping(uint256 => EventData) events
* mapping(uint256 => Ticket) tickets
* mapping(bytes32 => uint256[]) userTickets
* mapping(bytes32 => bool) usedPayments

## Core Functions

createEvent()

mintTicket(eventId, phoneHash, paymentId)

* require event active
* require not sold out
* require paymentId unused
* increment ticketsSold

markAsUsed(ticketId, phoneHash)

getUserTickets(phoneHash)
getTicket(ticketId)
getEvent(eventId)

Privacy:
phoneHash = keccak256(phone + eventId + SECRET_SALT)

---

# 4. Backend Architecture (Express)

Server responsibilities:

* OTP generation + console display
* Fake payment verification
* Smart contract interaction
* Session handling
* Serve user + admin pages

---

# 5. Authentication (Console OTP Mode)

POST /api/send-otp
Body:
{
"phone": "9876543210"
}

Backend:

* Generate 8-char secure OTP
* Hash and store temporarily
* Print OTP in server console

Example console log:
OTP for 9876543210: Ab9XkP2q

---

POST /api/verify-otp
Body:
{
"phone": "9876543210",
"otp": "Ab9XkP2q"
}

Backend:

* Verify OTP
* Issue JWT token

Frontend stores JWT.

---

# 6. Events

GET /api/events

Backend:

* Fetch events from smart contract
* Return active events

---

# 7. Booking Flow (Fake Payment)

Step 1: Initiate Booking

POST /api/book-ticket/initiate
Header: Authorization: Bearer <JWT>
Body:
{
"eventId": 1
}

Backend:

* Generate OTP
* Print in console

---

Step 2: Confirm Booking

POST /api/book-ticket/confirm
Header: Authorization: Bearer <JWT>
Body:
{
"eventId": 1,
"otp": "Ab9XkP2q",
"paymentId": "PAY123456"
}

Fake Payment Logic:

* Accept paymentId
* Hash paymentId
* Ensure not already used

Then:

1. Compute phoneHash
2. Compute paymentHash
3. Call mintTicket()

---

# 8. My Bookings

GET /api/my-bookings
Header: Authorization: Bearer <JWT>

Backend:

1. Compute phoneHash
2. getUserTickets(phoneHash)
3. Fetch ticket + event details
4. Return list

---

# 9. Entry Validation (Admin Panel)

Admin page served at:
/admin

QR contains:
{
"ticketId": 5
}

Step 1 – Initiate Entry
POST /api/entry/initiate
Body:
{
"ticketId": 5,
"phone": "9876543210"
}

Backend:

* Generate OTP
* Print in console

---

Step 2 – Confirm Entry
POST /api/entry/confirm
Body:
{
"ticketId": 5,
"phone": "9876543210",
"otp": "Ab9XkP2q"
}

Backend:

1. Verify OTP
2. Compute phoneHash
3. Call markAsUsed()
4. Return "Entry Granted"

---

# 10. App Pages (Served from Express Views)

## /login

* Enter phone
* Verify OTP

## /events

* List events
* Book button

## /checkout

* OTP verification
* Fake payment input
* Confirm booking

## /my-bookings

* Display active + used tickets
* QR generation button

## /admin

* QR scanner
* Phone input
* Entry validation

---

# 11. Security (MVP Level)

* 8-char cryptographic OTP
* OTP hashed in backend
* OTP expiry (5 min)
* Max 3 attempts
* PaymentId uniqueness enforced
* Event capacity enforced
* No raw phone stored on-chain

---

# 12. MVP Characteristics

* No wallet required
* No real SMS dependency
* No real payment dependency
* Blockchain-backed transparency
* Identity-bound ticketing
* Fully demo-safe
* Single Express server for everything

---

END OF MVP DOCUMENT
