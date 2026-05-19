# Algocode — API Gateway

> Single entry point for all client traffic in the AlgoCode online judge platform. Handles JWT authentication, role-based access control, and request routing to downstream microservices. No downstream service is exposed directly to the client.

---

## Table of Contents

- [Overview](#overview)
- [System Position](#system-position)
- [Why a Dedicated Gateway?](#why-a-dedicated-gateway)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Auth & RBAC Design](#auth--rbac-design)
- [API Reference](#api-reference)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Part of the AlgoCode Platform](#part-of-the-algocode-platform)

---

## Overview

In any microservices system, exposing each service directly to clients creates security, versioning, and coupling problems. The API Gateway solves this by acting as the single trusted entry point. Every request is authenticated here before it touches any downstream service.

This service is responsible for:
- Verifying JWT tokens on every protected request
- Enforcing role-based permissions before forwarding
- Managing user registration and login
- Structured logging of all inbound requests for traceability

---

## System Position

```
                        ┌──────────────────────────┐
                        │        Client             │
                        └──────────┬───────────────┘
                                   │ All requests
                                   ▼
                        ┌──────────────────────────┐
                        │      API Gateway          │
                        │  ┌────────────────────┐  │
                        │  │  Auth Middleware    │  │
                        │  │  (JWT Validation)  │  │
                        │  └────────────────────┘  │
                        │  ┌────────────────────┐  │
                        │  │  RBAC Middleware    │  │
                        │  │  (Role Check)      │  │
                        │  └────────────────────┘  │
                        └────┬──────────┬───────────┘
                             │          │
               ┌─────────────┘          └─────────────┐
               ▼                                       ▼
   ┌───────────────────┐                 ┌─────────────────────┐
   │   Problem Service │                 │  Evaluator Service  │
   └───────────────────┘                 └─────────────────────┘
```

---

## Why a Dedicated Gateway?

### Problem: Each service can't independently verify identity
In a microservices system, if every service handles auth independently, you get duplicated JWT logic, inconsistent role enforcement, and a much larger attack surface. A bug in one service's auth check exposes the whole system.

### Solution: Centralised auth at the edge
The gateway is the only service that ever touches user credentials or tokens. Once a request passes the gateway, downstream services trust it implicitly. This means:
- Auth logic lives in exactly one place — easier to audit and update
- Downstream services stay small and focused on their domain
- A compromised service can't forge identity — it never has the JWT secret

---

## Key Engineering Decisions

### 1. Stateless JWT authentication
No server-side session storage. The JWT carries the user's identity and roles as signed claims. This means:
- The gateway can scale horizontally without a shared session store
- Any gateway instance can verify any token with just the shared secret
- Token expiry is enforced at the cryptographic level, not a database lookup

### 2. Three-tier RBAC via a junction table
Roles are stored in a separate `Role` table with a `User_Role` junction table, not as a column on the `User` table. This means:
- A user can hold multiple roles simultaneously
- New roles can be added without schema changes to the User table
- The relationship is queryable and auditable

```
User ──── User_Role ──── Role
(1)         (M:M)        (1)
```

### 3. Repository pattern for database access
Controllers never touch the database directly. All DB operations go through a repository layer. This enforces separation of concerns and makes the data access logic independently testable.

```
Controller → Service → Repository → Sequelize/MySQL
```

### 4. Centralized error handling via AppError
All errors across the codebase are thrown as instances of a custom `AppError` class that carries an HTTP status code and a descriptive message. A single Express error middleware catches everything and formats consistent JSON responses — no ad-hoc `res.status(500).json(...)` scattered through handlers.

### 5. Winston structured logging
Every inbound request is logged with method, path, status, and duration. Errors are logged with stack traces. This makes distributed debugging tractable — you can correlate a failed request in the gateway logs with a job failure in the evaluator logs using a shared request ID.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Runtime | Node.js | Non-blocking I/O, large ecosystem |
| Framework | Express.js | Lightweight, middleware-friendly |
| Database | MySQL | Relational model fits user/role relationships |
| ORM | Sequelize | Schema migrations, association management |
| Auth | JWT (jsonwebtoken) | Stateless, scalable |
| Password Hashing | bcrypt | Slow hashing resistant to brute-force |
| Logging | Winston | Structured logs, configurable transports |

---

## Project Structure

```
src/
├── config/
│   ├── database.js      # Sequelize connection + sync
│   ├── server.js        # Express app setup, middleware registration
│   └── logger.js        # Winston logger configuration
├── controllers/
│   ├── user.js          # Signup / signin request handlers
│   └── role.js          # Role creation and assignment
├── middlewares/
│   ├── auth.js          # JWT verification — attaches user to req
│   ├── adminOnly.js     # Guards admin-only routes
│   └── validate.js      # Request body validation
├── models/
│   ├── user.js          # Sequelize User model
│   ├── role.js          # Sequelize Role model
│   └── user_role.js     # Junction table (M:M)
├── repositories/
│   ├── user.js          # DB queries for user operations
│   └── role.js          # DB queries for role operations
├── routes/
│   └── v1/
│       ├── user.js      # /api/v1/user routes
│       └── role.js      # /api/v1/role routes
├── services/
│   ├── user.js          # Business logic: hash passwords, generate tokens
│   └── role.js          # Business logic: role validation, assignment
└── utils/
    ├── errors.js        # AppError class definition
    ├── enums.js         # Role constants (ADMIN, CUSTOMER, etc.)
    └── response.js      # Standardised JSON response formatters
```

---

## Auth & RBAC Design

### Token flow

```
POST /api/v1/user/signin
  → validate credentials
  → bcrypt.compare(password, hash)
  → jwt.sign({ userId, roles }, JWT_SECRET, { expiresIn: JWT_EXPIRY })
  → return { token }

Protected Route
  → Authorization: Bearer <token>
  → jwt.verify(token, JWT_SECRET)
  → attach user + roles to req.user
  → role middleware checks req.user.roles includes required role
  → forward to handler or return 403
```

### Roles

| Role | Access |
|---|---|
| `ADMIN` | Full access: create roles, assign roles, all problem operations |
| `CUSTOMER` | Submit solutions, view problems, view own submissions |

Roles are assigned via the `/api/v1/role/addRoleToUser` endpoint (Admin only). A user has no roles by default until explicitly assigned.

---

## API Reference

### Auth Routes — `/api/v1/user`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/signup` | Register a new user | No |
| `POST` | `/signin` | Login and receive a JWT | No |

**POST /signup**
```json
// Request
{
  "username": "dhruv",
  "email": "dhruv@example.com",
  "password": "StrongPassword123"
}

// Response 201
{
  "success": true,
  "message": "User created successfully",
  "data": { "id": 1, "username": "dhruv", "email": "dhruv@example.com" }
}
```

**POST /signin**
```json
// Request
{
  "email": "dhruv@example.com",
  "password": "StrongPassword123"
}

// Response 200
{
  "success": true,
  "message": "Login successful",
  "data": { "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
}
```

---

### Role Routes — `/api/v1/role` *(Admin only)*

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/` | Create a new role | Yes — ADMIN |
| `POST` | `/addRoleToUser` | Assign a role to a user | Yes — ADMIN |

---

## Getting Started

### Prerequisites

- Node.js v18+
- MySQL running locally or via Docker

### Installation

```bash
git clone https://github.com/Algocode-dev/API_Gateway.git
cd API_Gateway
npm install
```

### Database Setup

```bash
# MySQL must be running. Sequelize will sync tables automatically on first run.
# Create the database manually first:
mysql -u root -p -e "CREATE DATABASE algocode_auth;"
```

### Run

```bash
# Development (with hot reload)
npm run dev

# Production
npm start
```

---

## Environment Variables

Create a `.env` file in the root:

```env
PORT=3000

# MySQL
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=yourpassword
DB_NAME=algocode_auth

# JWT
JWT_SECRET=your_jwt_secret_min_32_chars
JWT_EXPIRY=24h
```

---

## Part of the AlgoCode Platform

This service is one of four in the AlgoCode distributed system:

| Service | Repo | Description |
|---|---|---|
| **API Gateway** | **You are here** | Auth (JWT + RBAC), routing, entry point |
| Problem Service | [AlgoCode-problem-service](https://github.com/Algocode-dev/AlgoCode-problem-service) | Problem and test case management |
| Evaluator Service | [Algocode-Evaluator_Service](https://github.com/Algocode-dev/Algocode-Evaluator_Service) | Sandboxed code execution engine |
| Notification Service | [Notification-service](https://github.com/Algocode-dev/Notification-service) | Async event-driven result delivery |****
