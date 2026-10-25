# KO2 Currency & Weather Hub

![Build](https://github.com/ko2javier/currency-data-hub/actions/workflows/deploy.yml/badge.svg)
![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.3.5-6DB33F?logo=springboot&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-19-DD0031?logo=angular)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Tests](https://img.shields.io/badge/tests-15%20passed-brightgreen)
![Coverage](https://img.shields.io/badge/coverage-50%25-yellow?logo=jacoco)
![License](https://img.shields.io/badge/license-MIT-blue)

Real-time currency exchange and weather dashboard backed by three independently deployable Spring Boot microservices. JWT authentication, Redis token blacklist, a 4-level cache strategy, and full observability with Prometheus + Grafana — all running in production on a Hetzner VPS via Docker Compose and GitHub Actions CI/CD.

![KO2 Platform Dashboard](dashboard.png)

---

## Architecture

```mermaid
graph TD
    A["Angular 19<br/>Vercel · hub.ko2-oreilly.com"] -->|"HTTPS + JWT"| CF

    subgraph VPS ["Hetzner VPS CX23 · Docker Compose"]
        CF["Cloudflare SSL · Nginx :80"] --> GW
        GW["API Gateway :7000<br/>Spring Cloud Gateway + WebFlux<br/>JWT validation · Redis blacklist · routing"]
        GW -->|"/auth/**"| AUTH["Auth Service :4000<br/>Spring Boot · JWT HS256 · BCrypt · RBAC"]
        GW -->|"/weather/** /currency/**"| API["API Service :5000<br/>Spring Boot · 4-level cache<br/>JUnit 5 · 50% JaCoCo coverage"]
        AUTH -.->|"user.login"| KFK
        API -.->|"currency.query · weather.query"| KFK
        KFK["Kafka :9092 · KRaft mode<br/>no ZooKeeper · 7-day retention"]
        KFK --> EVT["ko2-kafka-events :6000<br/>audit-group + alert-group consumers"]
        PROM["Prometheus :9090<br/>scrapes /actuator/prometheus"] --> GW
        PROM --> AUTH
        PROM --> API
        GRAF["Grafana :3000<br/>pre-provisioned dashboard"] --> PROM
        PROM --> EVT
        RD[("Redis :6379<br/>self-hosted container<br/>AOF · 128 MB cap")]
    end

    EVT --> DB

    AUTH --> DB[("MySQL<br/>Aiven Cloud")]
    API --> DB
    AUTH --> RD
    API --> RD
    GW --> RD
    API -->|"cache miss"| ER["open.er-api.com"]
    API -->|"cache miss"| OM["Open-Meteo API"]
```

**Authenticated request flow:**
1. Angular sends `Authorization: Bearer <token>` → `api.ko2-oreilly.com`
2. Cloudflare terminates TLS → Nginx proxies to Gateway `:7000`
3. Gateway checks Redis blacklist + validates JWT signature
4. Gateway injects `X-User-Name` / `X-User-Roles` headers downstream
5. API Service reads headers via `HeaderAuthFilter` — no JWT re-validation, no secret sharing

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | Angular 19 — standalone components, RxJS | Standalone API removes NgModules boilerplate; signals-ready |
| Gateway | Spring Cloud Gateway + WebFlux | Non-blocking reactive I/O; centralises auth so downstream services stay JWT-unaware |
| Auth Service | Spring Boot 3 · JWT HS256 · BCrypt | Isolated auth boundary; token blacklist in Redis covers logout without server state |
| API Service | Spring Boot 3 · JPA · RestTemplate | Business logic and cache isolated from auth; independently deployable |
| Events | Apache Kafka 3.7 (KRaft) | Audit logging off the request path — producers fire-and-forget, the consumer replays from its last offset after a restart. KRaft mode drops the ZooKeeper dependency |
| Database | MySQL on Aiven | Managed cloud — automatic backups, no DBA overhead for a solo-deployed project |
| Cache | Redis 7.2 — self-hosted container | Two distinct concerns: token blacklist (Gateway + Auth) and data TTL (API Service). Moved off managed hosting in October 2026: it sits on the same Compose network as its only three clients, so it needs no public port, and both workloads are TTL-bound — nothing in it is worth paying to have replicated |
| Observability | Prometheus + Grafana | Production metrics — request rate, p95 latency, JVM heap, error rates per service |
| Infrastructure | Hetzner VPS CX23 | Best cost/performance ratio in Europe for this workload |
| Deployment | Docker Compose + GitHub Actions | Right-sized for four services on one host; push to a repo's main branch → auto-deploy |
| SSL | Cloudflare Flexible | Edge TLS termination + DDoS protection at zero operational cost |

---

## Key Technical Decisions

### 1. Gateway owns authentication — services are JWT-unaware
The Gateway validates JWT once and injects plain HTTP headers downstream. No JWT library, no shared secret in API/Auth services. Changing the auth mechanism only requires touching one service.

### 2. Redis token blacklist with self-expiring keys
On logout, the token's remaining TTL is used as the Redis key expiry. The blacklist cleans itself — no cron jobs, no table growth. Checking a Redis key adds ~1 ms vs. a synchronous DB read on every authenticated request.

### 3. 4-level cache with stale fallback
```
Request → Redis (10 min TTL) → MySQL (< 10 min old) → External API → stale MySQL record
```
The stale fallback prevents hard 503s when upstream APIs (Open-Meteo, ExchangeRate) are temporarily unavailable. Redis handles hot sub-millisecond reads; MySQL provides persistence across Redis restarts.

### 4. Audit events over Kafka instead of synchronous calls

Auth and API publish audit events (`user.login`, `currency.query`, `weather.query`) to Kafka and forget about them. `ko2-kafka-events` consumes with two independent groups — `audit-group` persists to `audit_events`, `alert-group` watches for rate-limit conditions and writes `alert_events`.

Nothing on the request path waits for audit logging, and the consumer can be down or redeployed without losing events: Kafka replays from the last committed offset. The broker runs in **KRaft mode**, so there is no ZooKeeper to operate, with a 7-day retention and a healthcheck the consumer waits on before starting.

### 5. Reactive gateway, blocking services
Spring Cloud Gateway runs on WebFlux (non-blocking). The two downstream services run on WebMVC (blocking/thread-per-request). This is intentional: the Gateway's job is routing at high concurrency — WebFlux fits. The business logic services are I/O-bound with simple CRUD; WebMVC is simpler to test and reason about.

---

## Observability

Prometheus scrapes `/actuator/prometheus` from all four KO2 services every 15 seconds — the three on the request path plus `ko2-kafka-events`, which exposes it on its own management port `:6001` — and from an independently-deployed sibling project on the same VPS: [`llm-gateway-resilience`](https://github.com/ko2javier/llm-gateway-resilience) (a Groq LLM gateway with Resilience4j, port 8085 — it runs as its own standalone container outside this Compose stack, attached to the Compose network so Prometheus can scrape it by container name). Grafana is auto-provisioned on startup with pre-built dashboards for both.

![Grafana Dashboard](docs/images/grafana.png)

**KO2 dashboard panels:**
- HTTP requests/sec per service
- HTTP latency p95 per service
- 4xx / 5xx error rate
- JVM heap memory used
- JVM live threads
- Service UP/DOWN status

**LLM Gateway Resilience dashboard panels:**
- Circuit breaker open/closed state (`groqApi`, `weatherApi`)
- Retry outcomes
- HTTP requests/sec and average latency
- JVM heap/threads

**Consumer lag is exported since October 2026.** `ko2-kafka-events` had a scrape job and exported `spring_kafka_listener_seconds_*` — how long the listener takes to handle a message, which is not lag. The Kafka **client** metrics were absent because `KafkaConfig` builds its own `DefaultKafkaConsumerFactory` and Spring Boot attaches a `MicrometerConsumerListener` only to the factory it auto-configures. The listener is now wired by hand, and a test in that repo fails if it is removed. The series to alert on is **`kafka_consumer_fetch_manager_records_lag_max`**: the per-partition `..._records_lag` only appears once lagging partitions are assigned, while `_max` exists from the client's first fetch. A Grafana panel for it is still to be built.

**Access in production:**
- Grafana: [api.ko2-oreilly.com/grafana](https://api.ko2-oreilly.com/grafana/) (login: `user` / `user123`) — served over HTTPS as a subpath of the Gateway's own domain, no separate subdomain needed
- Prometheus: not published. It listens on the server's loopback interface only, so the UI is reached over an SSH tunnel when needed; Grafana queries it over the Compose network. Every service in the stack is bound the same way: Nginx and Cloudflare are the only way in from outside.

---

## Live Demo

| | URL |
|---|---|
| Frontend | [hub.ko2-oreilly.com](https://hub.ko2-oreilly.com) |
| Swagger UI | [api.ko2-oreilly.com/webjars/swagger-ui](https://api.ko2-oreilly.com/webjars/swagger-ui/index.html) |

**Test credentials:**

| Username | Password | Role |
|---|---|---|
| `user` | `user123` | ROLE_USER |
| `admin` | `admin123` | ROLE_ADMIN |

---

## Testing

| | |
|---|---|
| Framework | JUnit 5 + Mockito |
| Tests passing | 15 |
| Coverage | 50.3% instruction coverage (JaCoCo) |

**Covered:** `WeatherService`, `CurrencyService`, `WeatherController`, `CurrencyController` — Redis cache hit, DB cache hit/expired, external API call, stale fallback, 404 city-not-found.

**Not yet covered:** `WeatherClient`, `CurrencyClient` (external HTTP), `HeaderAuthFilter` — next step via Testcontainers.

```bash
./gradlew test jacocoTestReport
# Report: build/reports/jacoco/test/html/index.html
```

---

## Related Repositories

| Repo | Description |
|---|---|
| [ko2-platform-frontend](https://github.com/ko2javier/ko2-platform-frontend) | Angular 19 SPA — dashboard, JWT auth flow, i18n (ES/EN/DE) |
| [auth-currency-data-hub](https://github.com/ko2javier/auth-currency-data-hub) | Auth Service :4000 — login, logout, JWT issuance, RBAC (USER / ADMIN / SUPERADMIN) |
| [currency-data-hub](https://github.com/ko2javier/currency-data-hub) | API Service :5000 — weather + currency endpoints, 4-level cache, 15 tests |
| [api-gateway-currency-data-hub](https://github.com/ko2javier/api-gateway-currency-data-hub) | Gateway :7000 — reactive routing, JWT validation, Redis blacklist |
| [ko2-kafka-events](https://github.com/ko2javier/ko2-kafka-events) | Events :6000 — Kafka consumers for audit and rate-limit alerting |

---

## Local Setup

<details>
<summary>Run with Docker Compose</summary>

### Prerequisites
- Docker + Docker Compose
- `.env` file with required variables (see `.env.example`)

### Required environment variables

```env
JWT_SECRET=your_256bit_secret
DB_URL=jdbc:mysql://your-mysql-host:3306/ko2db
DB_USERNAME=your_user
DB_PASSWORD=your_password
REDIS_PASSWORD=your_redis_password
REDIS_URL=redis://default:your_redis_password@redis:6379
AUTH_SERVICE_URL=http://auth-service:4000
API_SERVICE_URL=http://api-service:5000
GATEWAY_URL=http://localhost:7000
GRAFANA_PASSWORD=admin
```

### Start

```bash
git clone https://github.com/ko2javier/server-infrastructure.git
cd server-infrastructure/portfolio-backend
cp .env.example .env   # fill in your values
docker compose up -d
```

| Service | URL |
|---|---|
| Gateway + Swagger | `http://localhost:7000/webjars/swagger-ui/index.html` |
| Prometheus | `http://localhost:9090` |
| Grafana | `http://localhost:3000` |

</details>

---

## CI/CD

Each microservice repo has its own GitHub Actions workflow. On push to its main branch — `master` for the three request-path services, `main` for `ko2-kafka-events` — all four run the same three gated stages:

1. **tests** — JUnit 5 + Testcontainers against real dependencies (MySQL and Redis for the request-path services; MySQL and a real Kafka broker for the consumer). A red test stops everything below.
2. **build-and-push** — the image is built in Actions and pushed to `ghcr.io`, tagged both `latest` and the commit SHA.
3. **deploy** — SSH into the VPS (`script_stop: true`, so the first failing command ends the job), `git pull` *this* repo for the Compose file, then `<SERVICE>_TAG=<sha> docker compose pull && up -d --no-deps <service>`, and wait for a `200` from the service's actuator before reporting success.

**The images are built in Actions, not on the VPS.** Until October 2026 every push ran `docker compose build` inside the server: a `gradle bootJar` there took three and a half minutes and competed for the RAM that is this machine's real limit, and a failed build left the repo already pulled to the new commit with the old container still serving. Deploys now pin the commit SHA instead of `latest`, so the running container states which code it serves and a rollback is the same command with the previous SHA. Each service reads its own tag variable — `AUTH_TAG`, `API_TAG`, `GATEWAY_TAG`, `KAFKA_EVENTS_TAG` — defaulting to `latest`, and each Compose service keeps its `build:` block as a fallback for bringing it up by hand.

`--no-deps` keeps a service deploy from touching `redis` or `kafka`, which belong to this repo rather than to the service being deployed.

**Application ports are published on `127.0.0.1` only** (`4000`, `5000`, `6000`, `6001`, `7000`). Services reach each other over the Compose network by name, Prometheus scrapes them the same way, and the only thing that needs them on the host is the deploy health check. From outside, the entry point is nginx.

The workflows call `docker compose` (v2). The VPS also has the abandoned v1 at `/usr/bin/docker-compose`, which crashes with `KeyError: 'ContainerConfig'` when recreating a container whose image was built by a modern Docker — the fault that silently broke every KO2 Hub deploy between May and October 2026. A shim at `/usr/local/bin/docker-compose` forwards any remaining v1 call to v2.

Prometheus, Grafana and the Kafka broker are started once with `docker compose up -d prometheus grafana kafka redis` and persist via named volumes (`prometheus_data`, `grafana_data`, `kafka_data`, `redis_data`). `ko2-kafka-events` waits on the broker's healthcheck before starting, so the broker must be up first.

**`prometheus.yml` is bind-mounted as a single file**, so it is bound to an inode. `git pull` replaces the file rather than editing it, which leaves the running container reading the old one — and `POST /-/reload` then returns 200 after reloading stale config. Changing it requires `docker compose up -d --force-recreate prometheus`, not a reload.

---

## License

MIT
