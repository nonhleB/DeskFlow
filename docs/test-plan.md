# DeskFlow Test Plan

**Sprint day:** Day 2 | **Skill evidenced:** TEST | **Status:** Draft v1
**Stack under test:** Node.js, Express 4, Mongoose 8 (MongoDB), JWT auth, express-validator

## 1. Objectives and scope

Establish automated test coverage for the DeskFlow API so that changes can be merged with confidence and run in CI (Day 7).

**In scope:** auth middleware, RBAC, error handling, ticket creation, ticket status transitions, auth login, all HTTP endpoints.
**Out of scope:** frontend, load/performance testing, third-party library internals.

## 2. Tools and approach

| Concern | Choice | Reason |
|---|---|---|
| Test runner | Jest | Built-in mocking and coverage (`jest --coverage`) |
| HTTP testing | Supertest | Exercises the real Express app without opening a port |
| Database (integration) | `mongodb-memory-server` | Isolated, no external DB needed locally or in CI |
| Database (unit) | `jest.mock` on Mongoose models | Fast, tests logic only |

New dev dependencies: `jest`, `supertest`, `mongodb-memory-server`.
New scripts: `test`, `test:unit`, `test:integration`, `test:coverage`.

## 3. Testability issues to fix first (Day 3 prerequisites)

| # | Issue | Fix |
|---|---|---|
| T1 | `server.js` calls `connectDB()` and `app.listen()` at import, so tests cannot load the app cleanly | Move app setup to `app.js` (exports `app`, no DB connect, no listen). `server.js` imports `app`, connects and listens |
| T2 | Routes return 501 stubs and do not use the controllers | Wire `authRoutes` and `ticketRoutes` to `authController` and `ticketController` |
| T3 | No `test` script or test dependencies | Add Jest, Supertest, memory server and scripts |
| T4 | `seed.js` referenced by `npm run seed` is not in the repo | Add it, or remove the script. Tests create users via the model instead |
| T5 | No register endpoint, so test users cannot be created over HTTP | Create users directly with `User.create` in test setup |
| T6 | `authorize()` exists but is not applied to any route (`PUT /api/tickets/:id` is meant to be Admin-only) | Apply `authorize('Admin')` when wiring routes |

## 4. Endpoint audit

Current state is from reading `server.js`, `routes/` and `middleware/`. Controllers and models are still to be reviewed (marked *verify*).

| Method | Path | Auth | Role | Current behaviour | Existing tests | Risk |
|---|---|---|---|---|---|---|
| GET | `/api/health` | None | Any | 200 JSON | None | Low |
| POST | `/api/auth/login` | None | Any | 501 stub | None | **High** |
| POST | `/api/tickets` | Bearer JWT | Employee | 501 stub | None | **High** |
| GET | `/api/tickets` | Bearer JWT | Employee (own) / Admin (all) | 501 stub | None | **High** |
| GET | `/api/tickets/:id` | Bearer JWT | Owner or Admin | 501 stub | None | **High** |
| PUT | `/api/tickets/:id` | Bearer JWT | Admin | 501 stub, no role check | None | **High** |
| ANY | unknown route | None | Any | 404 via `notFound` | None | Low |

| Middleware / unit | File | Existing tests | Risk |
|---|---|---|---|
| `protect` | `middleware/auth.js` | None | **High** |
| `authorize` | `middleware/auth.js` | None | **High** |
| `errorHandler`, `notFound` | `middleware/errorHandler.js` | None | Medium |
| `ApiError`, `asyncHandler` | `middleware/` | None | Low |
| Auth controller | `controllers/authController.js` | None | *verify* |
| Ticket controller | `controllers/ticketController.js` | None | *verify* |

## 5. Unit vs integration split

**Unit tests** (mocked models, no database, no HTTP): test one function's logic in isolation. Fast, run on every change.
**Integration tests** (Supertest plus in-memory MongoDB): test the full request path: routing, middleware, controller, model, error handler.

## 6. Unit test cases

### 6.1 `protect` (auth middleware)
| ID | Case | Expected |
|---|---|---|
| U-AUTH-01 | No `Authorization` header | 401 `ApiError`, "No token provided" |
| U-AUTH-02 | Header without `Bearer ` prefix | 401 `ApiError` |
| U-AUTH-03 | Malformed or wrongly signed token | `JsonWebTokenError` forwarded to `next` |
| U-AUTH-04 | Expired token | `TokenExpiredError` forwarded to `next` |
| U-AUTH-05 | Valid token, user no longer exists | 401 `ApiError`, "User no longer exists" |
| U-AUTH-06 | Valid token, user exists | `req.user` set, `next()` called with no error |

### 6.2 `authorize` (RBAC)
| ID | Case | Expected |
|---|---|---|
| U-RBAC-01 | User role is in the allowed list | `next()` called |
| U-RBAC-02 | User role not in the allowed list | 403 `ApiError` |
| U-RBAC-03 | No `req.user` | 403 `ApiError`, role "unknown" |
| U-RBAC-04 | Multiple roles allowed, user matches the second | `next()` called |

### 6.3 Ticket creation (`ticketController`, *verify exact names*)
| ID | Case | Expected |
|---|---|---|
| U-TKT-01 | Valid payload | Ticket created with `createdBy = req.user._id`, default status, 201 |
| U-TKT-02 | Missing required field (e.g. title) | Validation error, 400 |
| U-TKT-03 | Client tries to set `status` or `createdBy` in the body | Ignored or rejected |

### 6.4 Ticket status transitions (*verify allowed statuses in `models/Ticket.js`*)
| ID | Case | Expected |
|---|---|---|
| U-STS-01 | Each valid transition (e.g. Open to In Progress to Resolved) | Status updated |
| U-STS-02 | Invalid status value | 400 |
| U-STS-03 | Invalid transition (e.g. Closed back to Open), if the rules forbid it | 400 |
| U-STS-04 | Ticket id does not exist | 404 |

### 6.5 Auth controller
| ID | Case | Expected |
|---|---|---|
| U-LOG-01 | Correct email and password | Returns signed JWT containing user id |
| U-LOG-02 | Wrong password | 401, generic message (no user enumeration) |
| U-LOG-03 | Unknown email | 401, same message as U-LOG-02 |
| U-LOG-04 | Missing email or password | 400 |

### 6.6 Error handling
| ID | Case | Expected |
|---|---|---|
| U-ERR-01 | `CastError` | 400, "Invalid value for field" |
| U-ERR-02 | Mongoose `ValidationError` | 400, joined messages |
| U-ERR-03 | Duplicate key (`code 11000`) | 400, names the field |
| U-ERR-04 | `JsonWebTokenError` / `TokenExpiredError` | 401, matching messages |
| U-ERR-05 | Unknown error | 500, `console.error` called |
| U-ERR-06 | `NODE_ENV=development` | Response includes `stack`; otherwise it does not |
| U-ERR-07 | `notFound` | Forwards 404 `ApiError` with method and URL |
| U-ERR-08 | `asyncHandler` with a rejecting function | Error passed to `next` |

## 7. Integration test cases (Supertest)

| ID | Request | Expected |
|---|---|---|
| I-HLT-01 | `GET /api/health` | 200, `success: true` |
| I-404-01 | `GET /api/does-not-exist` | 404 JSON |
| I-LOG-01 | `POST /api/auth/login` valid credentials | 200, token returned |
| I-LOG-02 | `POST /api/auth/login` wrong password | 401 |
| I-LOG-03 | `POST /api/auth/login` missing fields | 400 |
| I-TKT-01 | `POST /api/tickets` no token | 401 |
| I-TKT-02 | `POST /api/tickets` invalid token | 401 |
| I-TKT-03 | `POST /api/tickets` Employee, valid body | 201, ticket owned by the user |
| I-TKT-04 | `POST /api/tickets` invalid body | 400 |
| I-TKT-05 | `GET /api/tickets` Employee | Only own tickets |
| I-TKT-06 | `GET /api/tickets` Admin | All tickets |
| I-TKT-07 | `GET /api/tickets/:id` owner | 200 |
| I-TKT-08 | `GET /api/tickets/:id` other Employee | 403 or 404 (*verify intended behaviour*) |
| I-TKT-09 | `GET /api/tickets/:id` Admin | 200 |
| I-TKT-10 | `GET /api/tickets/not-an-objectid` | 400 (CastError) |
| I-TKT-11 | `GET /api/tickets/:id` valid id that does not exist | 404 |
| I-RBAC-01 | `PUT /api/tickets/:id` Employee | 403 |
| I-RBAC-02 | `PUT /api/tickets/:id` Admin, valid status | 200, status changed |
| I-RBAC-03 | `PUT /api/tickets/:id` Admin, invalid status | 400 |
| I-RBAC-04 | Token for a deleted user | 401 |

## 8. Priorities and schedule

| Priority | Cases | Planned |
|---|---|---|
| P1 (must) | U-AUTH, U-RBAC, U-TKT, U-STS, I-LOG, I-TKT-01 to 06, I-RBAC | Day 3 |
| P2 (should) | U-LOG, U-ERR, remaining I-TKT, I-HLT, I-404 | Day 4 |
| P3 (nice) | Edge cases found while testing | Day 4 if time allows |

Day 3: fix T1 to T6, write P1 tests. Day 4: P2 tests, coverage run, fix gaps.

## 9. Coverage targets

Collected with `jest --coverage` (`collectCoverageFrom`: `controllers/**`, `middleware/**`, `routes/**`, `app.js`).

| Area | Lines target |
|---|---|
| `middleware/` | 90% |
| `controllers/` | 80% |
| Overall | 75% |

`config/db.js` and `server.js` are excluded (infrastructure only).

## 10. Test data and environment

- `NODE_ENV=test` (already suppresses Morgan logging in `server.js`).
- `JWT_SECRET` set in the test setup file.
- Each integration suite starts the in-memory MongoDB, seeds one Admin and two Employees, and clears collections between tests.
- Tokens are generated in a helper using `jsonwebtoken` and the test secret.

## 11. Entry and exit criteria

**Entry:** testability fixes T1 to T3 merged.
**Exit:** all P1 and P2 cases pass, coverage targets met, coverage report and test output captured as evidence, PR merged.

## 12. Risks

| Risk | Mitigation |
|---|---|
| Controller logic differs from route comments | Review controllers before writing tests and update this plan |
| Memory server download fails in CI | Cache the binary or use a Mongo service container in GitHub Actions |
| Tests become coupled to implementation | Assert on HTTP responses and persisted data, not internals |
