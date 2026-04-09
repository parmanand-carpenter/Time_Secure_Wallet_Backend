# TimeDelay Wallet — Backend

Node.js backend service for the Time-Delay Smart Contract Wallet. It listens to on-chain events from both **Sepolia** and **XHAVIC L2**, stores transaction state in PostgreSQL, and sends email alerts to wallet owners at queue time and execution time.

---

## How It Works

1. **Blockchain Listener** — connects to both chains and watches for `TransactionQueued`, `TransactionCancelled`, and `TransactionExecuted` events from all registered wallet contracts.
2. **Job Queue (BullMQ + Redis)** — each event is pushed as a job into a Redis queue for reliable async processing.
3. **Worker** — processes jobs: saves transaction state to PostgreSQL and sends the appropriate email (cancel alert or execute reminder).
4. **REST API** — Express endpoints for the frontend to register wallets and query transaction history.

---

## Architecture

```
Blockchain (Sepolia / XHAVIC)
        │
        ▼
Blockchain Listener       ←─── WebSocket (Sepolia) / HTTP polling (XHAVIC)
        │
        ▼
BullMQ Queue (Redis)
        │
        ▼
Transaction Worker
   ├── Save to PostgreSQL (Prisma)
   └── Send Email (Nodemailer)
        │
        ├── Queue email  → Cancel link (on TransactionQueued)
        └── Execute email → Execute link (on delay expiry)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js (ESM) |
| API Server | Express.js |
| ORM | Prisma v5 |
| Database | PostgreSQL |
| Queue | BullMQ |
| Cache / Queue Store | Redis (ioredis) |
| Email | Nodemailer (SMTP) |
| Web3 | ethers.js v6 |

---

## Prerequisites

- [Node.js](https://nodejs.org/) v18 or higher
- PostgreSQL database
- Redis instance
- SMTP credentials (Gmail, SendGrid, etc.)

---

## Installation

```bash
cd timedalaywalletbackend
npm install
```

---

## Configuration

Create a `.env` file inside `timedalaywalletbackend/`:

```env
# Sepolia
RPC_URL=wss://sepolia.infura.io/ws/v3/YOUR_KEY
FACTORY_ADDRESS=0x0301327589710f6A9978cbf77D02C914902b76A3

# XHAVIC Testnet
XHAVIC_RPC_URL=https://testrpc.xhaviscan.com
XHAVIC_FACTORY_ADDRESS=0xE4FC0db39138dd457C9b4b4DA73Bf3e19cec7F37

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/timedelay_db

# Redis
REDIS_URL=redis://localhost:6379

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=your_app_password
EMAIL_FROM=your@gmail.com

# Server
PORT=3000
FRONTEND_URL=http://localhost:3000
```

> Never commit your `.env` file.

---

## Database Setup

```bash
# Generate Prisma client
npm run db:generate

# Push schema to database (creates tables)
npm run db:push
```

To open Prisma Studio (visual database browser):

```bash
npm run db:studio
```

---

## Running the Server

### Development (auto-restart on file change)

```bash
npm run dev
```

### Production

```bash
npm start
```

On startup, the server will:
1. Connect to PostgreSQL
2. Start the BullMQ worker
3. Start blockchain listeners for Sepolia and XHAVIC
4. Start the Express API on the configured `PORT`

---

## API Endpoints

### Register a Wallet

```
POST /api/wallet/register
```

Body:
```json
{
  "walletAddress": "0x...",
  "ownerAddress": "0x...",
  "email": "user@example.com"
}
```

Called by the frontend after `createWallet()` is executed on-chain. Links the wallet to an email for notifications.

---

### Get Wallets by Owner

```
GET /api/wallet/:ownerAddress
```

Returns all wallet contracts owned by the given EOA address.

---

### Get Transactions by Wallet

```
GET /api/transactions/:walletAddress
```

Returns all transactions for a wallet, ordered by most recent first.

---

### Cancel Transaction (Web Page)

```
GET /cancel/:walletAddress/:txId
```

Serves an HTML page with MetaMask integration. Users click "Cancel" to sign and broadcast `cancelTransaction(txId)` directly from their browser.

---

### Execute Transaction (Web Page)

```
GET /execute/:walletAddress/:txId
```

Serves an HTML page with MetaMask integration. Users click "Execute" to sign and broadcast `executeTransaction(txId)` directly from their browser.

---

### Health Check

```
GET /health
```

Returns `{ "status": "ok", "timestamp": "..." }`.

---

## Email Notifications

The backend sends two types of emails automatically:

| Email | Trigger | Content |
|-------|---------|---------|
| **Cancel Alert** | `TransactionQueued` event detected | Transaction details + Cancel button link |
| **Execute Reminder** | Delay timer expires (scheduled via BullMQ) | Transaction details + Execute button link |

Both emails contain a one-click link that opens the cancel or execute page directly in the browser.

---

## Project Structure

```
timedalaywalletbackend/
├── src/
│   ├── index.js                  # Entry point — wires all services together
│   ├── config/
│   │   ├── env.js                # Env validation and export
│   │   ├── db.js                 # Prisma client
│   │   └── redis.js              # ioredis connection
│   ├── listener/
│   │   └── blockchainListener.js # Event listener for Sepolia (WSS) and XHAVIC (HTTP poll)
│   ├── queue/
│   │   └── transactionQueue.js   # BullMQ queue + job adder functions
│   ├── worker/
│   │   └── transactionWorker.js  # Job processor — DB writes + email dispatch
│   ├── services/
│   │   └── emailService.js       # Nodemailer cancel/execute email templates
│   ├── api/
│   │   └── routes.js             # Express routes + cancel/execute HTML pages
│   └── utils/
│       └── logger.js             # Colored console logger
├── prisma/
│   └── schema.prisma             # Wallet + Transaction models (PostgreSQL)
├── package.json
└── .env                          # Environment variables (not committed)
```

---

## Database Schema

**Wallet**

| Field | Type | Description |
|-------|------|-------------|
| walletAddress | String (PK) | Contract address (lowercase) |
| ownerAddress | String | EOA owner address |
| email | String | Notification email |
| createdAt | DateTime | Registration time |

**Transaction**

| Field | Type | Description |
|-------|------|-------------|
| txId | Int | On-chain transaction ID |
| walletAddress | String (FK) | Linked wallet contract |
| toAddress | String | Recipient address |
| value | String | Amount in wei |
| token | String | Token address (zero address = native) |
| executeAfter | Int | Unix timestamp — earliest execution time |
| status | String | PENDING / CANCELLED / EXECUTED |
| executeMailSent | Boolean | Prevents duplicate execute emails |

---

## License

MIT
