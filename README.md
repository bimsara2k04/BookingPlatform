# EN2H Booking Platform — REST API

A clean, interview-ready backend REST API for managing **services** and **customer bookings**, built with NestJS, TypeScript, PostgreSQL, TypeORM, JWT authentication, and Swagger documentation.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Environment Variables](#environment-variables)
- [Database Setup](#database-setup)
- [Migration Steps](#migration-steps)
- [Running the App](#running-the-app)
- [API Documentation](#api-documentation)
- [API Endpoints](#api-endpoints)
- [Authentication Flow](#authentication-flow)
- [Business Rules](#business-rules)
- [Assumptions](#assumptions)
- [Future Improvements](#future-improvements)

---

## Project Overview

The EN2H Booking Platform API enables:

- **Admins** to manage services (create, update, delete) via JWT-protected endpoints
- **Customers** to browse services and book appointments without creating an account
- **Admins** to view and manage booking statuses
- **Customers** to self-cancel their bookings using just the booking ID

---

## Tech Stack

| Technology | Purpose |
|---|---|
| [NestJS](https://nestjs.com/) | Application framework |
| [TypeScript](https://www.typescriptlang.org/) | Language |
| [PostgreSQL](https://www.postgresql.org/) | Database |
| [TypeORM](https://typeorm.io/) | ORM & migrations |
| [JWT / Passport](https://www.passportjs.org/) | Authentication |
| [bcrypt](https://github.com/kelektiv/node.bcrypt.js) | Password hashing |
| [class-validator](https://github.com/typestack/class-validator) | DTO validation |
| [Swagger](https://swagger.io/) | API documentation |

---

## Project Structure

```
backend/src/
├── auth/                    # JWT authentication
│   ├── dto/                 # RegisterDto, LoginDto
│   ├── guards/              # JwtAuthGuard
│   ├── strategies/          # JwtStrategy
│   ├── auth.controller.ts
│   ├── auth.module.ts
│   └── auth.service.ts
├── bookings/                # Booking management
│   ├── dto/                 # CreateBookingDto, UpdateBookingStatusDto
│   ├── entities/            # Booking entity
│   ├── enums/               # BookingStatus enum
│   ├── bookings.controller.ts
│   ├── bookings.module.ts
│   └── bookings.service.ts
├── common/
│   └── filters/             # HttpExceptionFilter (global error shape)
├── config/
│   ├── data-source.ts       # TypeORM CLI DataSource
│   └── typeorm.config.ts    # NestJS runtime TypeORM config
├── migrations/              # TypeORM migration files
├── services/                # Service management
│   ├── dto/                 # CreateServiceDto, UpdateServiceDto
│   ├── entities/            # Service entity
│   ├── services.controller.ts
│   ├── services.module.ts
│   └── services.service.ts
├── users/                   # User data access
│   ├── entities/            # User entity
│   ├── users.module.ts
│   └── users.service.ts
├── app.module.ts
└── main.ts
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

| Variable | Description | Default |
|---|---|---|
| `DB_HOST` | PostgreSQL host | `localhost` |
| `DB_PORT` | PostgreSQL port | `5432` |
| `DB_USERNAME` | PostgreSQL username | `postgres` |
| `DB_PASSWORD` | PostgreSQL password | — |
| `DB_NAME` | Database name | `entwoh_booking` |
| `JWT_SECRET` | Secret key for JWT signing | — |
| `JWT_EXPIRES_IN` | JWT expiry duration | `1d` |

> **Security**: Never commit `.env` to version control. The `.gitignore` already excludes it.

---

## Database Setup

### Prerequisites
- PostgreSQL 13+ (uses `gen_random_uuid()` — no extensions required)

### Create the database

```bash
# Using psql
psql -U postgres -h localhost
CREATE DATABASE entwoh_booking;
\q
```

Or using pgAdmin: create a new database named `entwoh_booking`.

---

## Migration Steps

```bash
# Run existing migrations (creates all tables)
npm run migration:run

# Generate a new migration after entity changes
npm run migration:generate -- src/migrations/MyMigrationName

# Revert the last applied migration
npm run migration:revert
```

> Migrations use `src/config/data-source.ts` which reads from `.env` automatically.

---

## Running the App

```bash
# Install dependencies
npm install

# Development mode (with watch)
npm run start:dev

# Production mode
npm run build
npm run start:prod
```

The server starts at: **http://localhost:3000**  
All routes are prefixed with `/api/v1`

---

## API Documentation

Interactive Swagger UI: **http://localhost:3000/api**

### Using Swagger for authenticated requests:
1. Call `POST /api/v1/auth/register` to create an admin user
2. Call `POST /api/v1/auth/login` to get a JWT token
3. Click the **Authorize** button in Swagger and enter: `Bearer <your_token>`

---

## API Endpoints

### Auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/v1/auth/register` | No | Register a new admin user |
| POST | `/api/v1/auth/login` | No | Login and receive JWT token |

### Services

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/v1/services` | Yes (JWT) | Create a new service |
| GET | `/api/v1/services` | No | List all active services |
| GET | `/api/v1/services/search/:title` | No | Search services by title (case-insensitive) |
| GET | `/api/v1/services/:id` | No | Get service details |
| PATCH | `/api/v1/services/:id` | Yes (JWT) | Update a service |
| DELETE | `/api/v1/services/:id` | Yes (JWT) | Delete a service |

### Bookings

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/v1/bookings` | No | Create a booking (public) |
| GET | `/api/v1/bookings` | Yes (JWT) | List all bookings |
| GET | `/api/v1/bookings/:id` | Yes (JWT) | Get booking details |
| PATCH | `/api/v1/bookings/:id/status` | Yes (JWT) | Update booking status |
| PATCH | `/api/v1/bookings/:id/cancel` | No | Cancel a booking (customer) |

---

## Authentication Flow

```
POST /auth/register
  Body: { "email": "admin@example.com", "password": "StrongPass123" }
  → 201: { "id": "...", "email": "admin@example.com" }

POST /auth/login
  Body: { "email": "admin@example.com", "password": "StrongPass123" }
  → 200: { "accessToken": "eyJ..." }

All protected routes:
  Header: Authorization: Bearer eyJ...
```

---

## Business Rules

1. **Past date prevention**: Bookings cannot be created with a date in the past (HTTP 400)
2. **Active service only**: Can only book an active service; inactive services return HTTP 400
3. **Duplicate slot prevention**: Each `(serviceId, bookingDate, bookingTime)` combination is unique (HTTP 409)
4. **Status transition guard**: A `CANCELLED` booking cannot be changed to `COMPLETED` (HTTP 409)
5. **Customer self-cancel**: Customers can cancel via `PATCH /bookings/:id/cancel` using only the booking ID — no auth required
6. **Password security**: Passwords are hashed with bcrypt (cost factor 12); the same error is returned for "user not found" and "wrong password" to prevent user enumeration

---

## Assumptions

1. **Single admin role**: All authenticated users have the same permissions — no role-based access control
2. **Booking date format**: Date is stored as a plain `DATE` column (no timezone); `bookingDate` should be sent as `YYYY-MM-DD`
3. **Booking time format**: Time is validated via regex `HH:MM` (24-hour) before being stored as a `TIME` column
4. **Hard delete**: Deleting a service permanently removes it (and cascades to related bookings via FK)
5. **Public booking read**: Customers need to retain their booking ID to cancel — there is no customer account system
6. **No soft-delete**: The `isActive` flag on services is used for visibility only, not as a soft-delete mechanism (DELETE is a hard delete)

---

## Future Improvements

- [ ] **Role-based access control** (RBAC): Separate `ADMIN` and `STAFF` roles
- [ ] **Refresh token** rotation for more secure JWT session management
- [ ] **Email notifications**: Send confirmation/cancellation emails via SendGrid or Nodemailer
- [ ] **Pagination**: Add cursor-based or offset pagination to GET list endpoints
- [ ] **Rate limiting**: Add `@nestjs/throttler` to prevent abuse on public endpoints
- [ ] **Soft delete**: Implement TypeORM `@DeleteDateColumn()` for services so bookings history is preserved
- [ ] **Customer accounts**: Let customers register to view their own booking history
- [ ] **Calendar integration**: Add availability checking — e.g. max concurrent bookings per time slot
- [ ] **Unit & integration tests**: Add Jest unit tests for services/service and e2e tests with `supertest`
- [ ] **Docker Compose**: Add a `docker-compose.yml` to spin up Postgres + the API together
- [ ] **CI/CD pipeline**: GitHub Actions for lint, build, test, and deploy
