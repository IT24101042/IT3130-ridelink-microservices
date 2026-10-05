# Ride Management Service: RideLink

Ride requests, driver assignment and the ride lifecycle. This is the integrating service: it calls the Driver & Vehicle Service and the Fare & Payment Service over synchronous REST.

**Owner:** Member 3 (fill in). **Port:** 8082. **Database:** PostgreSQL `ride_db` (owned only by this service).

## Prerequisites
JDK 21+, Maven, PostgreSQL (`CREATE DATABASE ride_db;`). Driver service (8082) and Fare service (8084) running for the full workflow.

## Configuration (see `.env.example`; never commit real values)
| Variable | Purpose                               |
|---|---------------------------------------|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | PostgreSQL                            |
| `JWT_SECRET` | Identical to the other three services |
| `DRIVER_SERVICE_URL` | default `http://localhost:8081`       |
| `FARE_SERVICE_URL` | default `http://localhost:8083`       |

## Run / test
```bash
mvn spring-boot:run
mvn test
```
Swagger UI: http://localhost:8082/swagger-ui.html

## Endpoints
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/rides` | PASSENGER | Request a ride: fare estimate, nearest eligible driver, assignment |
| GET | `/rides/mine` | Authenticated | Own rides (ADMIN: all) |
| GET | `/rides/{id}` | Passenger, driver or ADMIN | Ride details |
| GET | `/rides/{id}/history` | Passenger, driver or ADMIN | Status history |
| PATCH | `/rides/{id}/accept` | Assigned DRIVER | ASSIGNED to ACCEPTED |
| PATCH | `/rides/{id}/start` | Assigned DRIVER | ACCEPTED to IN_PROGRESS |
| PATCH | `/rides/{id}/complete` | Assigned DRIVER | IN_PROGRESS to COMPLETED, final fare, payment |
| PATCH | `/rides/{id}/cancel` | Passenger, driver or ADMIN | Before the trip starts |

## Lifecycle
REQUESTED, ASSIGNED, ACCEPTED, IN_PROGRESS, COMPLETED. CANCELLED is allowed from REQUESTED, ASSIGNED and ACCEPTED. COMPLETED and CANCELLED are final.

## Assignment rule (documented)
Ask the Driver service for eligible drivers near the pickup (nearest first), pick the first, and mark it ON_TRIP so it cannot be double-booked. Released back to AVAILABLE on completion or cancellation.

## Interservice communication
| Interaction | Style | Why |
|---|---|---|
| Ride to Driver: eligible drivers, set availability | Synchronous REST | The answer is needed immediately to decide the assignment |
| Ride to Fare: estimate, final fare | Synchronous REST | The passenger and the completion response need the amount now |
| Ride to Fare: record payment | Synchronous REST | Result stored on the ride; failure is recorded as `paymentStatus=FAILED`, the ride stays COMPLETED |

Outbound calls use a short-lived service token (role ADMIN, subject `ride-service`) signed with the shared secret, plus connect/read timeouts. Remote failures return 503 (unavailable) or 502 (rejected).

## Negative scenarios
No available driver: 409. Invalid transition: 409. Wrong driver or non-participant: 403. Unknown ride: 404. Invalid input: 400. Missing/invalid token: 401. Driver or Fare service down: 503.
# RideLink: Backend Microservices for a Ride-Sharing Platform

IT3130 Application Development, Group Assignment. Backend only: Swagger UI and the Postman collection are the official interfaces.

| **Release tag:** [v1.0.0]

## Service owners

| Service | Primary owner | Port | Database | Folder |
|---|---|------|---|---|
| Account Service | [Member 1 name] | 8081 | PostgreSQL `account_db` | `account-service/` |
| Driver & Vehicle Service | [Member 2 name] | 8081 | MongoDB `driver_db` | `driver-service/` |
| Ride Management Service | [Member 3 name] | 8082 | PostgreSQL `ride_db` | `ride-service/` |
| Fare & Payment Service | [Member 4 name] | 8083 | PostgreSQL `fare_db` | `fare-payment-service/` |

Each service owns its own database. No service reads or writes another service's data; they talk only through REST APIs.

## Architecture in one minute

- **Account** is the only service that issues JWTs. The other three validate the token locally with the same shared secret.
- **Ride Management** orchestrates the booking. It calls **Driver & Vehicle** (eligible drivers, driver availability) and **Fare & Payment** (estimate, final fare, simulated payment) over synchronous REST.
- Ride Management uses a short-lived service token (role ADMIN, subject `ride-service`) for those outbound calls.
- Details, diagrams and the communication comparison are in the technical report.

## Repository layout

```
ridelink/
  account-service/
  driver-service/
  ride-service/
  fare-payment-service/
  postman/                  exported collection and environment
  .github/workflows/        one CI workflow per service
  README.md                 this file
```

## Prerequisites

- JDK 21 or newer, Maven
- PostgreSQL (for Account, Ride and Fare)
- MongoDB (for Driver), for example `docker run -d -p 27017:27017 --name mongo mongo:7`
- Postman (optional, for the collection)

## Configuration

Every service has an `.env.example`. Copy it to `.env` or set the same variables in your IDE run configuration. **Never commit real values.**

| Variable | Used by | Meaning |
|---|---|---|
| `JWT_SECRET` | all four | Long random string. **Must be identical in all four services.** There is no default. |
| `JWT_EXPIRATION_MS` | Account | Token lifetime, default 86400000 (24 hours) |
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | Account, Ride, Fare | PostgreSQL connection |
| `MONGO_URI` | Driver | MongoDB connection string |
| `DRIVER_SERVICE_URL` | Ride | default `http://localhost:8082` |
| `FARE_SERVICE_URL` | Ride | default `http://localhost:8084` |
| `PAYMENT_MAX_AMOUNT` | Fare | Simulated payment limit, default 50000 |
| `INCLUDE_ERROR_CAUSE` | Ride | Dev switch. Keep `false` for the demo. |

Create the PostgreSQL databases once:

```sql
CREATE DATABASE account_db;
CREATE DATABASE ride_db;
CREATE DATABASE fare_db;
```

MongoDB creates `driver_db` automatically on first write.

## Start-up order

1. PostgreSQL and MongoDB running
2. Account Service (8081)
3. Driver & Vehicle Service (8082)
4. Fare & Payment Service (8084)
5. Ride Management Service (8083)

In each service folder:

```bash
mvn spring-boot:run
```

## Swagger UI (OpenAPI documentation)

| Service | Swagger UI                            |
|---|---------------------------------------|
| Account | http://localhost:8080/swagger-ui.html |
| Driver & Vehicle | http://localhost:8081/swagger-ui.html |
| Ride Management | http://localhost:8082/swagger-ui.html |
| Fare & Payment | http://localhost:8083/swagger-ui.html |

OpenAPI JSON is at `/v3/api-docs` on each service. Use the **Authorize** button and paste a JWT from login.

## Sample test data (fictional)

| Role | Email | Password |
|---|---|---|
| Passenger | `passenger1@example.com` | `password123` |
| Passenger (second) | `passenger2@example.com` | `password123` |
| Driver | `driver1@example.com` | `password123` |
| Admin | `admin1@example.com` | `password123` |

Create them with `POST /accounts/register`, then log in with `POST /accounts/login`. [Describe here how the admin account is created if public ADMIN registration is disabled.]

Sample coordinates (Colombo): pickup Fort `6.9271, 79.8612`, destination Mount Lavinia `6.8389, 79.8653`, driver location `6.9275, 79.8620`.

## Running the tests

Unit tests need no database or running service. In each service folder:

```bash
mvn test
```

Integrated tests: import `postman/RideLink.postman_collection.json` and `postman/RideLink.postman_environment.json` into Postman, set the four service URLs in the environment, start all four services, and run the collection in order (setup, driver preparation, fare, ride lifecycle, negative scenarios).

## Main workflow (manual)

1. Register and log in as passenger and driver.
2. Driver: `POST /drivers`, `PUT /drivers/{id}/vehicle`, `PUT /drivers/{id}/location`, `PATCH /drivers/{id}/availability` with `AVAILABLE`.
3. Passenger: `POST /fares/estimate`, then `POST /rides`.
4. Driver: `PATCH /rides/{id}/accept`, `/start`, `/complete`.
5. Passenger: `GET /payments/ride/{rideId}` and `GET /payments/{id}/receipt`.

## Ride lifecycle

`REQUESTED` to `ASSIGNED` to `ACCEPTED` to `IN_PROGRESS` to `COMPLETED`. `CANCELLED` is allowed from `REQUESTED`, `ASSIGNED` and `ACCEPTED`. Invalid transitions return 409.

## Negative scenarios demonstrated

No available driver (409), invalid status transition (409), wrong role or non-participant (403), missing token (401), invalid input (400), unknown resource (404), failed simulated payment (ride stays COMPLETED with `paymentStatus` FAILED), Fare or Driver service unavailable (503).

## Version control workflow

- `main` always holds an integrated, demonstrable version and changes only through pull requests.
- `develop` is the integration branch.
- Work happens on feature branches named `feature/<service>-<topic>`, for example `feature/driver-eligible-query`.
- Every pull request needs at least one review from another member and a passing CI check.
- Commit messages are short and describe one change. The assessed version carries the release tag `[v1.0.0]`.

## Continuous integration

Each service has a GitHub Actions workflow in `.github/workflows/` that builds it and runs its tests (`mvn -B clean verify`) on pushes and pull requests to `main` and `develop`.

## Service READMEs

Each service folder contains its own README with endpoints, rules and negative scenarios:
[Account](account-service/README.md), [Driver & Vehicle](driver-service/README.md), [Ride Management](ride-service/README.md), [Fare & Payment](fare-payment-service/README.md).

## Known limitations

Documented in Section 9 of the technical report: no atomic driver reservation, no distributed transactions, synchronous payment recording without retries, and a shared symmetric JWT secret.
