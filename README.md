# API Gateway

> **Unified entry point** for the Car Rental System (CRS) microservice architecture.  
> Routes all client requests to the appropriate downstream service, and handles cross-origin requests.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Running the Gateway](#running-the-gateway)
- [Routing Configuration](#routing-configuration)
- [CORS Configuration](#cors-configuration)
- [Request Flow](#request-flow)
- [Adding New Routes](#adding-new-routes)

---

## Overview

The API Gateway is the **single ingress point** for all client traffic. The React frontend never calls individual microservices directly — every request goes through the gateway on port **8888**.

Responsibilities:
- **Request routing** — Matches URL paths and forwards to the correct downstream service
- **CORS handling** — Allows the frontend (`localhost:5173`) to make cross-origin requests with credentials
- **Protocol bridging** — Receives HTTP/1.1 from clients, forwards to reactive WebFlux backends

> This gateway uses **static route configuration** (no service registry/Eureka). Every service has a fixed hostname and port defined in `application.yml`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.2.2 |
| Gateway Engine | Spring Cloud Gateway 2023.0.0 (reactive, WebFlux-based) |
| Build | Maven |

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │       API Gateway (:8888)        │
                        │                                  │
  React Frontend  ─────►│  Route Matching (application.yml)│
  (:5173)               │                                  │
                        └──────────────┬───────────────────┘
                                       │
               ┌───────────────────────┼────────────────────┐
               │                       │                    │
               ▼                       ▼                    ▼
       iam-service              car-management         booking-service
          (:8080)                   (:8081)                (:8082)
     /auth/**                  /vehicles/**           /bookings/**
     /admin/**                 /simulator/**          /drivers/**
     /roles/**                                        /payments/**
     /permissions/**                                  /dashboard/**
```

---

## Getting Started

### Prerequisites

- Java 21+
- Maven 3.8+
- All downstream services running (iam-service, car-management, booking-service)

### Running the Gateway

```bash
cd api-gateway

# Run directly
mvn spring-boot:run

# Or build and run as JAR
mvn clean package -DskipTests
java -jar target/api-gateway-*.jar
```

The gateway starts on **port 8888**. Once running, all API calls from the frontend go to `http://localhost:8888/api/v1/...`.

> **Order matters**: Start the gateway only after all downstream services are up, otherwise early requests will receive `502 Bad Gateway`.

---

## Routing Configuration

All routes are defined in `src/main/resources/application.yml`. Each route entry specifies:
- `id` — unique route identifier
- `uri` — downstream service URL
- `predicates` — path pattern to match
- `filters` — `StripPrefix=0` (no path rewriting; the full path is forwarded as-is)

### Complete Route Table

| Route ID | Path Predicate | Upstream Service | Port |
|---|---|---|---|
| `iam-auth-route` | `/api/v1/auth/**` | IAM Service | `8080` |
| `iam-admin-users-route` | `/api/v1/admin/**` | IAM Service | `8080` |
| `iam-roles-route` | `/api/v1/roles/**` | IAM Service | `8080` |
| `iam-permissions-route` | `/api/v1/permissions/**` | IAM Service | `8080` |
| `car-vehicles-route` | `/api/v1/vehicles/**` | Car Management | `8081` |
| `car-simulator-route` | `/api/v1/simulator/**` | Car Management | `8081` |
| `booking-rentals-route` | `/api/v1/bookings/**` | Booking Service | `8082` |
| `booking-drivers-route` | `/api/v1/drivers/**` | Booking Service | `8082` |
| `booking-payments-route` | `/api/v1/payments/**` | Booking Service | `8082` |
| `booking-dashboard-route` | `/api/v1/dashboard/**` | Booking Service | `8082` |

### Route Configuration Example

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: iam-auth-route
          uri: http://localhost:8080
          predicates:
            - Path=/api/v1/auth/**
          filters:
            - StripPrefix=0

        - id: car-vehicles-route
          uri: http://localhost:8081
          predicates:
            - Path=/api/v1/vehicles/**
          filters:
            - StripPrefix=0
```

### Authentication Handling

The gateway itself **does not validate JWT tokens**. Token validation is delegated entirely to each downstream service. This is an intentional simplification — the `iam-service` handles all auth concerns including token introspection.

---

## CORS Configuration

The gateway is configured to allow cross-origin requests from the React development server:

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:5173"
            allowedMethods:
              - GET
              - POST
              - PUT
              - PATCH
              - DELETE
              - OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
```

| Setting | Value |
|---|---|
| Allowed Origins | `http://localhost:5173` (Vite dev server) |
| Allowed Methods | GET, POST, PUT, PATCH, DELETE, OPTIONS |
| Allowed Headers | All (`*`) |
| Allow Credentials | `true` — required for sending `Authorization` headers |

> **Production note**: Replace `allowedOrigins` with your deployed frontend domain (e.g., `https://your-domain.com`).

---

## Request Flow

A typical authenticated request from the frontend:

```
1. React app sends:
   GET http://localhost:8888/api/v1/bookings?page=0&size=10
   Headers: Authorization: Bearer eyJhbGci...

2. API Gateway receives the request.
   Matches route: booking-rentals-route (Path=/api/v1/bookings/**)

3. Gateway forwards to:
   GET http://localhost:8082/api/v1/bookings?page=0&size=10
   Headers: (all original headers forwarded including Authorization)

4. booking-service validates the JWT, processes the request, returns response.

5. Gateway returns the response to the React client.
```

---

## Adding New Routes

To expose a new endpoint from any backend service:

1. Open `src/main/resources/application.yml`
2. Add a new entry under `spring.cloud.gateway.routes`:

```yaml
- id: my-new-route
  uri: http://localhost:<service-port>
  predicates:
    - Path=/api/v1/<your-path>/**
  filters:
    - StripPrefix=0
```

3. Restart the gateway (`mvn spring-boot:run`).

> **Route ordering**: Spring Cloud Gateway evaluates routes in the order they are defined. Place more specific patterns **before** wildcard patterns to avoid incorrect matches.
