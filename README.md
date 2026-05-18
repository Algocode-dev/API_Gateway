# algocode-gateway

API Gateway and Authentication Service for the [AlgoCode](https://github.com/Dhruv7850) online judge platform. Acts as the single entry point for all client traffic — handles routing, user authentication, and role-based access control across the microservices architecture.

---

## Overview

In the AlgoCode system, every request from the client hits this service first. It validates identity, enforces permissions, and routes traffic to the appropriate downstream service (problem service, judge engine, notification service). No downstream service is exposed directly.

```
Client → algocode-gateway → algocode-problems
                          → algocode-judge
                          → algocode-notifications
```

---

## Features

- **JWT Authentication** — Stateless auth using signed JSON Web Tokens; no session storage required
- **Role-Based Access Control (RBAC)** — Three roles: `ADMIN`, `CUSTOMER`, and `FLIGHT_COMPANY`, each with distinct route permissions
- **User Management** — Full signup and signin flow with bcrypt password hashing
- **Request Validation Middleware** — Guards protected routes before forwarding to downstream services
- **Centralized Error Handling** — Custom `AppError` class ensures consistent error responses across all routes
- **Structured Logging** — Winston logger for request tracing and error diagnostics

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js |
| Framework | Express.js |
| Database | MySQL |
| ORM | Sequelize |
| Auth | JSON Web Tokens (jsonwebtoken) |
| Password Hashing | bcrypt |
| Logging | Winston |

---

## Project Structure

```
src/
├── config/         # Environment config, server setup, Winston logger
├── controllers/    # Request handlers — delegates to service layer
├── middlewares/    # Auth checks, request validation, admin guards
├── models/         # Sequelize models: User, Role, User_Role (junction)
├── repositories/   # Database access layer — Repository Pattern
├── routes/         # Versioned API routes (v1)
├── services/       # Business logic — password hashing, token generation
└── utils/          # Shared helpers, enums, response formatters
```

---

## Getting Started

### Prerequisites

- Node.js v18+
- MySQL running locally or via Docker

### Installation

```bash
git clone https://github.com/Dhruv7850/algocode-gateway.git
cd algocode-gateway
npm install
```

### Environment Variables

Create a `.env` file in the root:

```env
PORT=3000
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=yourpassword
DB_NAME=algocode_auth
JWT_SECRET=your_jwt_secret
JWT_EXPIRY=24h
```

### Run

```bash
# Development
npm run dev

# Production
npm start
```

---

## API Reference

### Auth Routes — `POST /api/v1/user`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/signup` | Register a new user | No |
| POST | `/signin` | Login and receive JWT | No |

### Role Routes — `POST /api/v1/role` *(Admin only)*

| Method | Endpoint | Description |
|---|---|---|
| POST | `/` | Create a new role |
| POST | `/addRoleToUser` | Assign a role to a user |

---

## Part of the AlgoCode Platform

This service is one of four in the AlgoCode microservices system:

| Service | Description |
|---|---|
| [algocode-gateway](https://github.com/Dhruv7850/algocode-gateway) | ← You are here — auth and routing |
| [algocode-judge](https://github.com/Dhruv7850/algocode-judge) | Sandboxed code execution engine |
| [algocode-problems](https://github.com/Dhruv7850/algocode-problems) | Problem and test case management |
| [algocode-notifications](https://github.com/Dhruv7850/algocode-notifications) | Async event-driven notification service |
