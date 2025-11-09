# 💸 PayPal-Like Payment System (Spring Boot Microservices).

A **Spring Boot microservices architecture** simulating a **PayPal-style payment system**, designed with **Spring Cloud Gateway**, **Spring Security**, **Kafka**, and **H2** databases.  
It supports **user authentication, wallet operations, transaction orchestration, notifications, and rewards**, communicating through REST and Kafka events.

---

## 🧠 High-Level Overview

This system models a **modern digital wallet ecosystem**, where services interact asynchronously and securely through Kafka and API Gateway.  
It demonstrates microservice principles such as **service separation, security, idempotency, and event-driven communication**.

---

## 🏗️ Services and Responsibilities

### 🛡️ API Gateway (port 8080)
- Routes requests to downstream microservices.  
- Validates JWT tokens for all paths except `/auth/**`.  
- Injects headers: `X-User-Id`, `X-User-Email`, and `X-User-Role`.  
- Implements **rate limiting** using `RedisRequestRateLimiter` (requires Redis at `localhost:6379`).

### 👤 User Service (port 8081)
- Handles **signup/login** operations.  
- Issues and validates **JWTs**.  
- Optional Spring Security filter for protected endpoints.

### 💰 Wallet Service (port 8088)
- Manages user wallets with:
  - Currency, total balance, available balance.  
- Supports operations: `createWallet`, `credit`, `debit`, `placeHold`, `captureHold`, `releaseHold`.  
- Includes `WalletHold` model with a scheduler for **hold expiry**.

### 💳 Transaction Service (port 8082)
- Orchestrates **fund transfers**:
  1. Place sender hold  
  2. Verify receiver wallet  
  3. Capture hold (debit sender)  
  4. Credit receiver  
- Persists transaction lifecycle: `PENDING → SUCCESS/FAILED`.  
- Publishes Kafka event `txn-initiated` on success.

### 🔔 Notification Service (port 8084)
- Consumes `txn-initiated` events.  
- Persists notifications for the sender with transaction details.

### 🏆 Reward Service (port 8089)
- Consumes `txn-initiated` events.  
- Creates `Reward` entries based on transaction amount.  
- Ensures **idempotency** using `transactionId`.

---

## 🔄 Data Flow (Typical Payment)

1. Client sends request to **API Gateway** with **Bearer JWT**.  
2. **Gateway** validates JWT → forwards to `transaction-service` with user headers.  
3. **Transaction-service**:
   - Validates caller vs senderId.  
   - Calls `wallet-service`:
     - `POST /hold` → reserve balance  
     - Verify receiver wallet  
     - `POST /capture` → debit sender  
     - `POST /credit` → credit receiver  
4. On success:
   - Saves transaction (`SUCCESS`).  
   - Emits Kafka event `txn-initiated`.  
5. **notification-service** and **reward-service** consume and persist their respective records.

---

## ⚙️ Tech Stack & Infrastructure

- **Spring Boot**, **Spring Cloud Gateway**, **Spring Security**  
- **Apache Kafka** + **Zookeeper** (via `docker-compose.yml`)  
- **H2 in-memory DB** with **JPA/Hibernate**  
  - `ddl-auto = update`  
  - `show-sql = true`  
- **Configuration:** YAML per service  
- **Communication:** REST (HTTP) + Kafka  

---

## 🚦 Gateway Route Configuration

| Route | Service | Auth Required | Rate Limit |
|-------|----------|---------------|-------------|
| `/auth/**` | user-service | ❌ | No |
| `/api/transactions/**` | transaction-service | ✅ | Yes |
| `/api/rewards/**` | reward-service | ✅ | Yes |
| `/api/notifications/**` | notification-service | ✅ | Yes |

---

## 🪄 Kafka Configuration

- **Topic:** `txn-initiated`  
- **Producer:** `transaction-service`  
- **Consumers:** `notification-service`, `reward-service`

---

## ⚠️ Risks & Improvements

- Reward-service Kafka consumer config cleanup (bootstrap servers spacing, trusted package path).  
- Redis dependency missing in `docker-compose.yml` (required by Gateway rate-limiter).  
- JWT secret duplicated across multiple services — should be centralized via **config server** or **environment variables**.  
- `transaction-service` uses raw JSON via `RestTemplate`; refactor to **Feign Client** or **WebClient** for better type safety.

---



## 🧩 Summary

This project demonstrates:
- Event-driven microservice communication (Kafka).  
- Secure API management using JWT and Gateway filters.  
- Transaction orchestration and idempotent reward handling.  
- Scalable, modular service architecture.

---
<img width="503" height="512" alt="image" src="https://github.com/user-attachments/assets/17acd080-f983-4117-a540-13ce2b591ea9" />

