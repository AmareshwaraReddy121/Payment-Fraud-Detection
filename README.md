# 🛡️ Fraud Detection System — MuleSoft API-Led Project


A complete end-to-end payment fraud detection system built using **MuleSoft's API-Led Connectivity** architecture. Real-time fraud scoring, asynchronous message processing, and automated risk team notifications.

![MuleSoft](https://img.shields.io/badge/MuleSoft-4.11.0-blue)
![Java](https://img.shields.io/badge/Java-17-orange)
![MySQL](https://img.shields.io/badge/MySQL-8.0-green)
![ActiveMQ](https://img.shields.io/badge/ActiveMQ-6.1.4-red)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Fraud Detection Rules](#-fraud-detection-rules)
- [API Endpoints](#-api-endpoints)
- [Setup & Installation](#️-setup--installation)
- [Configuration](#-configuration)
- [Running the Project](#️-running-the-project)
- [Testing](#-testing)
- [End-to-End Flow](#-end-to-end-flow)
- [Error Handling](#-error-handling)
- [Key Design Decisions](#-key-design-decisions)
- [Future Enhancements](#-future-enhancements)

---

## 🎯 Overview

This project implements a real-time payment fraud detection system using MuleSoft's three-layer API-led architecture. It accepts payment requests, applies fraud detection rules, asynchronously processes transactions via a message queue, persists data to a database, and notifies a risk team about blocked transactions via email.

### Key Features

- ✅ **3-layer API-led architecture** (Experience, Process, System APIs)
- ✅ **Asynchronous fraud scoring** via JMS queues
- ✅ **Priority-based message queueing** (high priority for fraud)
- ✅ **Real-time fraud rules engine**
- ✅ **Hourly risk team notifications** via email
- ✅ **CSV audit trail** of all blocked transactions
- ✅ **Centralized error handling** per API
- ✅ **MUnit test coverage**

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Customer / Mobile App                    │
└─────────────────────────────────────────────────────────────┘
                              ↓ POST /api/v1/payments
┌─────────────────────────────────────────────────────────────┐
│           🟦 Experience API (port 8089)                      │
│           Clean public contract — validates input            │
└─────────────────────────────────────────────────────────────┘
                              ↓ POST /api/process-payment
┌─────────────────────────────────────────────────────────────┐
│           🟩 Process API (port 8087)                         │
│           Fraud rules engine + JMS queueing                  │
│                                                              │
│   ┌──────────────────┐    ┌──────────────────┐               │
│   │  Receive Flow    │ →  │  ActiveMQ Queue  │               │
│   │  (Apply rules)   │    │  payments-queue  │               │
│   └──────────────────┘    └────────┬─────────┘               │
│                                    ↓                         │
│   ┌──────────────────┐    ┌──────────────────┐               │
│   │  Scheduler Flow  │    │   Worker Flow    │               │
│   │  (Hourly check)  │    │ (JMS Listener)   │               │
│   └────────┬─────────┘    └────────┬─────────┘               │
└────────────┼──────────────────────┼──────────────────────────┘
             ↓                      ↓
             ↓                  POST /api/payments
             ↓                      ↓
┌─────────────────────────────────────────────────────────────┐
│           🟨 System API (port 8085)                          │
│           DB operations + Email notifications                │
└─────────────────────────────────────────────────────────────┘
                  ↓                              ↓
        ┌──────────────────┐         ┌──────────────────┐
        │  MySQL Database  │         │      SMTP        │
        │  payment_db      │         │  (App Password)  │
        └──────────────────┘         └──────────────────┘
```

---

## 🛠 Tech Stack

| Component      | Technology              |
| -------------- | ----------------------- |
| Runtime        | Mule Server 4.11.0 EE   |
| IDE            | Anypoint Studio         |
| JDK            | OpenJDK 17              |
| Database       | MySQL 8.0               |
| Message Broker | ActiveMQ Classic 6.1.4  |
| Email          | SMTP (STARTTLS)         |
| API Spec       | RAML 1.0                |
| Testing        | MUnit 3.7 + Postman     |
| Build Tool     | Maven                   |

---

## 📁 Project Structure

```
fraud-detection-system/
├── fraud-detection-experience-api/       # Experience API (port 8089)
│   ├── src/main/mule/
│   │   ├── payment-experience-api.xml
│   │   └── global-error-handler.xml
│   ├── src/main/resources/api/
│   │   └── payment-experience-api.raml
│   ├── src/test/munit/
│   │   └── payment-experience-api-test-suite.xml
│   └── pom.xml
│
├── fraud-detection-process-api/          # Process API (port 8087)
│   ├── src/main/mule/
│   │   ├── payment-process-api.xml
│   │   └── global-error-handler.xml
│   ├── src/main/resources/
│   │   ├── api/payment-process-api.raml
│   │   └── config-dev.yaml
│   └── pom.xml
│
└── fraud-detection-system-api/           # System API (port 8085)
    ├── src/main/mule/
    │   ├── payment-system-api.xml
    │   └── global-error-handler.xml
    ├── src/main/resources/
    │   ├── api/payment-system-api.raml
    │   └── config-dev.yaml
    └── pom.xml
```

---

## 🔍 Fraud Detection Rules

The Process API applies these rules and **always overrides** the caller's `isRisk` field for security:

| Rule | Condition                  | Reason                       |
| ---- | -------------------------- | ---------------------------- |
| 1    | `amount > 50000`           | High amount transaction      |
| 2    | `country != cardCountry`   | Country mismatch             |
| 3    | `cardNumber in blacklist`  | Blacklisted card detected    |

**Blacklisted cards (sample):** maintained in the Process API DataWeave logic.

If any rule fires → status = `BLOCKED`, queue priority = 9, email alert triggered.
If no rules fire → status = `SUCCESS`, queue priority = 4.

---

## 🔌 API Endpoints

### Experience API (port 8089)

| Method | Endpoint              | Description                       |
| ------ | --------------------- | --------------------------------- |
| POST   | `/api/v1/payments`    | Submit a payment for processing   |

### Process API (port 8087)

| Method | Endpoint                  | Description                              |
| ------ | ------------------------- | ---------------------------------------- |
| POST   | `/api/process-payment`    | Internal endpoint — applies fraud rules  |

### System API (port 8085)

| Method | Endpoint                          | Description                          |
| ------ | --------------------------------- | ------------------------------------ |
| POST   | `/api/payments`                   | Insert payment record into DB        |
| GET    | `/api/payments`                   | Get all BLOCKED transactions         |
| POST   | `/api/notifications/email`        | Send fraud alert email               |

---

## ⚙️ Setup & Installation

### Prerequisites

- Anypoint Studio 7.x with Mule 4.11.0
- MySQL 8.0+ running on `localhost:3306`
- ActiveMQ Classic 6.1.4+ running on `localhost:61616`
- SMTP account with App Password configured (e.g., Gmail App Password)
- Java 17

### Step 1: Database setup

Create the database and table:

```sql
CREATE DATABASE payment_db;

USE payment_db;

CREATE TABLE payments (
    transaction_id VARCHAR(50) PRIMARY KEY,
    card_number VARCHAR(20),
    card_holder VARCHAR(100),
    amount DECIMAL(15,2),
    currency VARCHAR(5),
    country VARCHAR(5),
    card_country VARCHAR(5),
    merchant_id VARCHAR(20),
    email VARCHAR(255),
    status VARCHAR(20),
    reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Step 2: ActiveMQ setup

Download ActiveMQ Classic 6.1.4, then:

```bash
cd apache-activemq-6.1.4/bin
./activemq start
```

Web console: `http://localhost:8161` (default credentials: `admin / admin`)

### Step 3: SMTP App Password

1. Enable 2FA on your email account
2. Generate an App Password from your email provider's security settings
3. Save the password to use in `config-dev.yaml` (do **not** commit this)

### Step 4: CSV output folder

Create the folder for risk reports:

```powershell
New-Item -ItemType Directory -Force -Path "C:\fraud-reports"

# Create header line (Windows PowerShell)
[System.IO.File]::WriteAllText(
    "C:\fraud-reports\risk-transactions.csv",
    "Transaction ID,Card Holder,Amount,Currency,Country,Card Country,Merchant ID,Email,Status,Created At`r`n",
    (New-Object System.Text.UTF8Encoding($false))
)
```

### Step 5: Clone & import projects

```bash
git clone <your-repo-url>
cd fraud-detection-system
```

Import each API project in Anypoint Studio:
- **File → Import → Anypoint Studio Project from File System**

---

## 🔧 Configuration

Each project has a `config-dev.yaml` file in `src/main/resources/`.

> ⚠️ **IMPORTANT**: Never commit credentials. Use the `config-dev.example.yaml` template below and add your real `config-dev.yaml` to `.gitignore`.

### `config-dev.example.yaml` (template — safe to commit)

```yaml
http:
  host: "0.0.0.0"
  port: "8087"            # Use 8085 for System API, 8087 for Process API, 8089 for Experience API

db:
  host: "localhost"
  port: "3306"
  user: "<YOUR_DB_USER>"
  password: "<YOUR_DB_PASSWORD>"
  database: "payment_db"

jms:
  broker:
    url: "tcp://localhost:61616"
    user: "<YOUR_JMS_USER>"
    password: "<YOUR_JMS_PASSWORD>"
  queue:
    name: "payments-queue"

smtp:
  host: "smtp.example.com"
  port: "587"
  user: "<YOUR_SMTP_USER>"
  password: "<YOUR_SMTP_APP_PASSWORD>"
  from: "<YOUR_FROM_ADDRESS>"

risk:
  team:
    email: "<RISK_TEAM_EMAIL>"

file:
  output:
    path: "C:/fraud-reports"
```

### `.gitignore` entries (recommended)

```
# Local configuration with secrets
**/config-dev.yaml
**/config-prod.yaml

# Build output
target/

# IDE files
.idea/
.vscode/
*.iml

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
```

---

## ▶️ Running the Project

### Start order matters

1. Start **MySQL** service
2. Start **ActiveMQ** broker
3. Start **System API** (port 8085)
4. Start **Process API** (port 8087) — wait for System API to be fully up
5. Start **Experience API** (port 8089)

### In Anypoint Studio

Right-click each project → **Run As → Mule Application**

Wait for each console to show:

```
* Started app 'xxx-api'
```

---

## 🧪 Testing

### Postman test (happy path — clean transaction)

```http
POST http://localhost:8089/api/v1/payments
Content-Type: application/json

{
  "transactionId": "TEST_001",
  "cardNumber": "4567891234567890",
  "cardHolder": "Test User",
  "amount": 2500,
  "currency": "INR",
  "country": "IN",
  "cardCountry": "IN",
  "merchantId": "MERCH001",
  "email": "customer@example.com"
}
```

Expected: `202 Accepted` + DB row with `status=SUCCESS`

### Postman test (fraud scenario — high amount)

```json
{
  "transactionId": "FRAUD_001",
  "cardNumber": "4567891234567890",
  "cardHolder": "Fraud Test",
  "amount": 75000,
  "currency": "INR",
  "country": "IN",
  "cardCountry": "US",
  "merchantId": "MERCH001",
  "email": "customer@example.com"
}
```

Expected: `202 Accepted` + DB row with `status=BLOCKED` + email alert within 1 hour.

### Postman test (blacklisted card)

```json
{
  "transactionId": "FRAUD_002",
  "cardNumber": "4111111111111111",
  "cardHolder": "Blacklist Test",
  "amount": 100,
  "currency": "INR",
  "country": "IN",
  "cardCountry": "IN",
  "merchantId": "MERCH002",
  "email": "customer@example.com"
}
```

Expected: `BLOCKED` with reason "Blacklisted card detected"

### MUnit tests

In Anypoint Studio:
- Right-click test suite XML → **Run MUnit Test**

Test coverage:
- Happy path validation
- Error handler (Process API down → 503)
- APIKit error scenarios (400, 404, 415)

### Verify ActiveMQ

Open `http://localhost:8161/admin/queues.jsp` and check:
- Queue `payments-queue` exists
- Messages Enqueued/Dequeued counts increment with each request

### Verify MySQL

```sql
SELECT transaction_id, card_holder, amount, status, reason, created_at
FROM payments
ORDER BY created_at DESC
LIMIT 10;
```

---

## 🔄 End-to-End Flow

### Customer sends a fraud transaction:

```
1. POST /api/v1/payments (Experience API)
   → Validates customer input, forwards to Process API

2. POST /api/process-payment (Process API)
   → Applies fraud rules (amount > 50000? country mismatch? blacklist?)
   → Computes isRisk + reason
   → Publishes to ActiveMQ queue with priority 9
   → Returns 202 Accepted to customer

3. JMS Worker Flow (Process API)
   → Consumes message from queue
   → Builds payment record with status=BLOCKED
   → Calls System API to persist

4. POST /api/payments (System API)
   → Inserts record into MySQL

5. Scheduler Flow (Process API) — runs hourly
   → Calls GET /api/payments to fetch BLOCKED transactions
   → Filters to last hour
   → Appends each transaction to risk-transactions.csv
   → For each: calls POST /api/notifications/email

6. Email Send (System API)
   → Sends fraud alert email to risk team
   → Email contains: transactionId, cardHolder, amount, reason, timestamp
```

### Console log trail (successful run)

```
[EXP-API] Received payment request from customer: FRAUD_001
[EXP-API] Forwarded transaction: FRAUD_001
[PROC-API] Received Payment
[PROC-API] Published to queue
[Worker] Saving Payment
[System-API] Inserting Payment
[Worker] Transaction Saved
...
[Scheduler] Starting hourly risk transaction
[System-API] Listing Payments
[Scheduler] Wrote risk report to file
[Scheduler] Email sent for transaction: FRAUD_001
[Scheduler] Hourly report run completed
```

---

## 🚨 Error Handling

Each API uses a **centralized global error handler** (`global-error-handler.xml`) referenced by all flows.

### Error response format

```json
{
  "error": {
    "code": 503,
    "errorType": "Service Unavailable",
    "errorDescription": "Payment service is temporarily unavailable. Please try again."
  }
}
```

### Common error scenarios

| Scenario              | Error Type                    | HTTP Code | Customer Sees                            |
| --------------------- | ----------------------------- | --------- | ---------------------------------------- |
| Missing required field| APIKIT:BAD_REQUEST            | 400       | "Invalid request"                        |
| Wrong URL             | APIKIT:NOT_FOUND              | 404       | "Resource not found"                     |
| Wrong HTTP method     | APIKIT:METHOD_NOT_ALLOWED     | 405       | "Method not allowed"                     |
| Wrong Content-Type    | APIKIT:UNSUPPORTED_MEDIA_TYPE | 415       | "Content-Type must be application/json"  |
| Downstream API down   | HTTP:CONNECTIVITY             | 503       | "Service temporarily unavailable"        |
| ActiveMQ down         | JMS:CONNECTIVITY              | 503       | "Message queue unavailable"              |
| MySQL down            | DB:CONNECTIVITY               | 503       | "Database temporarily unavailable"       |
| SMTP down             | EMAIL:CONNECTIVITY            | 503       | "Email service unavailable"              |
| Any other             | ANY (catch-all)               | 500       | "Something went wrong"                   |

### Per-flow error handling strategy

| Flow Type            | Strategy              | Why                                                  |
| -------------------- | --------------------- | ---------------------------------------------------- |
| HTTP-facing flows    | `on-error-propagate`  | Return error response to caller                      |
| JMS Listener (Worker)| `on-error-continue`   | Acknowledge message, prevent infinite redelivery     |
| Scheduler            | `on-error-continue`   | Log and exit cleanly, retry on next schedule         |

---

## 🎯 Key Design Decisions

| Decision                                  | Rationale                                                                |
| ----------------------------------------- | ------------------------------------------------------------------------ |
| 3-layer API-led architecture              | Separation of concerns; each layer has a single responsibility           |
| Process API computes `isRisk`             | **Security**: callers cannot bypass fraud detection by claiming clean    |
| Asynchronous processing via JMS           | Customer response not blocked by fraud scoring; better UX                |
| JMS priority-based queueing               | Fraud transactions processed first (priority 9 vs 4)                     |
| Time-based filter (last hour) for scheduler | Natural deduplication — no overlap between hourly runs                |
| CSV file in APPEND mode                   | Audit trail of all blocked transactions over time                        |
| Single global error handler per API       | Consistent error responses; easier maintenance                           |
| Customer never sees "blocked" status      | Fraud handled internally; risk team manages via email                    |

---

## 🚀 Future Enhancements

- [ ] Replace polling scheduler with **event-driven** real-time alerts
- [ ] Add **Object Store** for distributed dedup logic
- [ ] Move from CSV to **dedicated audit table** in DB
- [ ] Implement **machine learning model** for fraud scoring
- [ ] Add **rate limiting** at Experience API
- [ ] Add **OAuth 2.0 / API key** authentication
- [ ] Deploy to **CloudHub** for production
- [ ] Add **dashboard UI** for risk team
- [ ] Integrate with **case management** systems
- [ ] Add **SMS notifications**
- [ ] Implement **circuit breakers** for downstream resilience
- [ ] Add **Prometheus + Grafana** monitoring

---

## 📊 Sample Test Cases

| Test                  | Input Highlights                          | Expected Result                  |
| --------------------- | ----------------------------------------- | -------------------------------- |
| Clean transaction     | amount=2500, country=IN, cardCountry=IN   | `SUCCESS`, no email              |
| High amount           | amount=75000                              | `BLOCKED`, email sent            |
| Country mismatch      | country=IN, cardCountry=US                | `BLOCKED`, email sent            |
| Blacklisted card      | cardNumber=4111111111111111               | `BLOCKED`, email sent            |
| Multiple rules        | amount=90000, country mismatch            | `BLOCKED`, combined reason       |
| Missing field         | No `cardNumber`                           | `400 Bad Request`                |
| Threshold edge        | amount=50000 (exactly)                    | `SUCCESS` (rule is `>`)          |
| Above threshold       | amount=50001                              | `BLOCKED`                        |

---

## 🔐 Security Notes

- **Never commit** `config-dev.yaml` or any file with real credentials.
- Use environment-specific config files (`config-dev.yaml`, `config-prod.yaml`) and add them to `.gitignore`.
- Rotate App Passwords periodically.
- For production, use a secret manager (HashiCorp Vault, AWS Secrets Manager, etc.) instead of YAML files.
- The Experience API intentionally **rejects** any `isRisk` or `reason` fields from callers to prevent fraud-detection bypass.

---

## 📝 License

This project is built for learning and demonstration purposes.

---

⭐ **If you found this project helpful, please star the repo!**
