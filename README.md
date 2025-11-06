🧩 High-Level Summary

A Spring Boot Microservices System simulating a “PayPal-like” payment flow, built with Spring Cloud Gateway, Spring Security, Kafka, and H2 databases.
The system handles user authentication, wallet management, transactions, notifications, and rewards, communicating over HTTP and Kafka.
All services run independently and integrate seamlessly through REST APIs and Kafka topics.

🏗️ Services and Responsibilities
1. api-gateway (port: 8080)

Routes incoming requests to downstream microservices.

Validates JWT for all routes except /auth/**.

Injects headers: X-User-Id, X-User-Email, and X-User-Role after token validation.

Implements rate limiting using Spring Cloud Gateway Redis RequestRateLimiter (requires Redis at localhost:6379).

2. user-service (port: 8081)

Handles user signup and login.

Issues and validates JWT tokens.

Includes optional Spring Security filter for authenticated requests.

3. wallet-service (port: 8088)

Manages user wallets, including:

Currency

Total balance

Available balance

Supports operations:

createWallet, credit, debit

placeHold, captureHold, releaseHold

Includes WalletHold model and hold lifecycle management.

A scheduler automatically expires holds after timeout.

4. transaction-service (port: 8082)

Orchestrates fund transfers between users.

Flow:

Places sender hold

Verifies receiver wallet

Captures hold (debit sender)

Credits receiver

Persists transaction lifecycle: PENDING → SUCCESS / FAILED.

Emits Kafka event txn-initiated on success for downstream consumers.

5. notification-service (port: 8084)

Consumes txn-initiated Kafka events.

Creates and stores a Notification record for the sender.

Notification contains transaction details such as amount and involved parties.

6. reward-service (port: 8089)

Consumes txn-initiated Kafka events.

Generates Reward records based on transaction amount.

Ensures idempotency using transactionId.

🔄 Typical Data Flow (Payment Transaction)

Client sends request to API Gateway with Bearer JWT.

Gateway validates token → forwards request to transaction-service with user headers.

Transaction-service:

Validates that caller = sender.

Calls wallet-service:

POST /hold → reserve sender’s balance.

Verify receiver’s wallet.

POST /capture → debit sender.

POST /credit → credit receiver.

On success:

Saves transaction (SUCCESS).

Publishes Kafka event txn-initiated.

notification-service and reward-service consume the event and persist related data.

⚙️ Tech & Infrastructure

Frameworks: Spring Boot, Spring Cloud Gateway, Spring Security

Messaging: Apache Kafka + Zookeeper (via docker-compose.yml)

Database: H2 (in-memory) using JPA/Hibernate

spring.jpa.hibernate.ddl-auto=update

spring.jpa.show-sql=true

Configuration: YAML per service

Communication: REST (HTTP) + Kafka

🧩 Gateway Route Configuration
Route	Destination Service	Auth	Rate Limit
/auth/**	user-service	❌	No
/api/transactions/**	transaction-service	✅	Yes
/api/rewards/**	reward-service	✅	Yes
/api/notifications/**	notification-service	✅	Yes
🪄 Kafka

Topic: txn-initiated

Producers: transaction-service

Consumers: notification-service, reward-service

🚨 Risks & Improvement Opportunities

Kafka consumer config (reward-service):
Needs cleanup — check bootstrap server string and trusted package configuration.

Redis dependency:
Required by gateway rate-limiter but missing in docker-compose.yml.

JWT secret duplication:
Secret is hardcoded in multiple services — should be centralized via environment variables or a config server.

Unsafe REST calls:
transaction-service uses raw JSON strings with RestTemplate; should migrate to Feign Client or WebClient for better type safety.
