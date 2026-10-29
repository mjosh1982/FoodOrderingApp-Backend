# FoodOrderingApp-Backend

Backend for a food-ordering application (in the style of Swiggy/Zomato), built with **Spring Boot 2.0.1**, **Spring Data JPA/Hibernate**, **PostgreSQL** and **Swagger 2**. The project follows the upGrad *FoodOrderingApp* course structure (`com.upgrad.FoodOrderingApp.*` packages).

> **Status: starter / skeleton.** The infrastructure is in place: the Maven multi-module build, API contracts (Swagger JSON), DB schema and seed data, the exception hierarchy, and password hashing and JWT helpers. **REST controllers, JPA entity mappings, DAOs and business services are not implemented yet.** The entity and service classes are empty, and all controller tests are commented out. The sections below describe both what exists and what the contracts expect.

---

## Table of contents

1. [Tech stack](#tech-stack)
2. [Repository layout](#repository-layout)
3. [Module overview](#module-overview)
4. [Request flow / architecture](#request-flow--architecture)
5. [API endpoints](#api-endpoints)
6. [Authentication](#authentication)
7. [Database schema](#database-schema)
8. [Seed data](#seed-data)
9. [Error handling](#error-handling)
10. [Tests](#tests)
11. [Configuration](#configuration)
12. [Getting started](#getting-started)
13. [Implementation status and known gaps](#implementation-status-and-known-gaps)

---

## Tech stack

| Concern            | Library / version                                            |
|--------------------|--------------------------------------------------------------|
| Framework          | Spring Boot 2.0.1.RELEASE (Spring 5.0.x)                     |
| Web / REST         | `spring-boot-starter-web` (embedded Tomcat)                  |
| Persistence        | `spring-boot-starter-data-jpa` (Hibernate 5.2), PostgreSQL driver 42.2.2 |
| Validation         | `spring-boot-starter-validation` (Hibernate Validator 6)     |
| API docs / codegen | Springfox Swagger2 + Swagger UI 2.6.1, `swagger-codegen-maven-plugin` 2.3.1 |
| Auth tokens        | `com.auth0:java-jwt` 3.4.0 (HMAC-512)                        |
| Utilities          | `commons-lang3` 3.4, Logback 1.1.7                           |
| DB scripting       | `sql-maven-plugin` 1.5, `properties-maven-plugin`            |
| Testing            | `spring-boot-starter-test` (JUnit 4, Mockito, MockMvc)       |
| Build              | Maven (multi-module), Java 8 language level                  |

## Repository layout

```
FoodOrderingApp-Backend/
├── pom.xml                         # Parent POM (packaging=pom), declares the 3 modules
├── FoodOrderingApp-api/            # Web layer: Spring Boot app, Swagger specs, controller tests
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/upgrad/FoodOrderingApp/api/
│       │   ├── FoodOrderingAppApiApplication.java   # main() entry point
│       │   └── config/SwaggerConfiguration.java     # Springfox Docket
│       ├── main/resources/
│       │   ├── application.yaml                     # server + datasource config
│       │   └── endpoints/*.json                     # Swagger 2.0 API contracts (7 files)
│       └── test/java/.../api/controller/*Test.java  # MockMvc controller tests (commented out)
├── FoodOrderingApp-service/        # Business layer: entities, services, exceptions, crypto/JWT
│   ├── pom.xml
│   └── src/main/java/com/upgrad/FoodOrderingApp/service/
│       ├── ServiceConfiguration.java                # @ComponentScan + @EntityScan
│       ├── businness/                               # services + security helpers
│       ├── common/                                  # ErrorCode, GenericErrorCode, UnexpectedException
│       ├── entity/                                  # JPA entities (currently empty stubs)
│       └── exception/                               # domain exceptions (13 classes)
├── FoodOrderingApp-db/             # Database module: schema + seed SQL, run via Maven profile
│   ├── pom.xml
│   └── src/main/resources/
│       ├── config/localhost.properties              # DB connection used by sql-maven-plugin
│       └── sql/tables.sql, sql/insert.sql
└── .idea/, *.iml, */target/        # IntelliJ metadata and build output (committed)
```

## Module overview

The parent `pom.xml` inherits from `spring-boot-starter-parent:2.0.1.RELEASE` and builds three modules. Dependencies point one way: **api → service**. The `db` module is standalone and only runs SQL scripts.

### `FoodOrderingApp-api` (web layer)

- **`FoodOrderingAppApiApplication`**: `@SpringBootApplication` entry point. It `@Import`s `ServiceConfiguration` so that service-layer beans and JPA entities get picked up.
- **`config/SwaggerConfiguration`**: enables Swagger 2 and registers a `Docket` that documents every handler in `com.upgrad.FoodOrderingApp.api.controller`. That package does not exist yet; controllers are meant to go there.
- **`resources/endpoints/*.json`**: Swagger 2.0 specs, one per domain (customer, address, category, item, order, payment, restaurant). These specs are the **API contract**.
- **Code generation**: `swagger-codegen-maven-plugin` runs one execution per spec with `generateApis=false` and `generateModels=true`. On every build it produces request/response DTOs in `com.upgrad.FoodOrderingApp.api.model` under `target/generated-sources`, for example `SignupCustomerRequest`, `LoginResponse`, `SaveOrderRequest`, `RestaurantDetailsResponse` and `ErrorResponse`. Controllers are expected to use these generated models instead of hand-written DTOs.

### `FoodOrderingApp-service` (business layer)

- **`ServiceConfiguration`**: `@ComponentScan("com.upgrad.FoodOrderingApp.service")` and `@EntityScan("com.upgrad.FoodOrderingApp.service.entity")`.
- **`businness/`** (the package name is spelled this way in the code):
  - `PasswordCryptographyProvider` (`@Component`): PBKDF2WithHmacSHA512, 1000 iterations, 64-bit key, 32-byte `SecureRandom` salt. `encrypt(password)` returns `[base64Salt, hexHash]`. The static `encrypt(password, salt)` recomputes the hash so a login attempt can be checked.
  - `JwtTokenProvider`: takes a secret and signs JWTs with HMAC-512. Claims are issuer `https://FoodOrderingApp.io`, audience = customer UUID, a random key id, issued-at and expires-at. Not a Spring bean; services create it with `new JwtTokenProvider(password)`.
  - `CustomerService`, `AddressService`, `PaymentService`: **empty stubs**. The tests also expect `CategoryService`, `ItemService`, `RestaurantService` and `OrderService`.
- **`entity/`**: `CustomerEntity`, `AddressEntity`, `StateEntity`, `PaymentEntity` are **empty stubs**. They are meant to map the tables in `tables.sql`. The tests also expect `CustomerAuthEntity`, `CategoryEntity`, `ItemEntity`, `RestaurantEntity`, `OrderEntity`, `CouponEntity` and similar.
- **`common/`**: `ErrorCode` interface, `GenericErrorCode` enum (`GEN-001`), and `UnexpectedException` (a runtime exception whose message comes from the error code).
- **`exception/`**: checked domain exceptions. Each one carries a `code` and an `errorMessage`. See [Error handling](#error-handling).

### `FoodOrderingApp-db` (database module)

- Reads `config/localhost.properties` (host, port, DB name, credentials) through `properties-maven-plugin`.
- The `setup` Maven profile binds `sql-maven-plugin` to `process-resources`. It runs `tables.sql` and then `insert.sql` against `jdbc:postgresql://localhost:5432/restaurantdb`.

## Request flow / architecture

The intended layering, as implied by the specs, tests and package structure:

```
HTTP client
   │  (JSON; Authorization: Basic … for login, Bearer <JWT> for protected calls)
   ▼
api.controller.*Controller        ← to be implemented; uses generated api.model.* DTOs
   │  (maps DTO ⇄ entity, maps exceptions → ErrorResponse via a @ControllerAdvice)
   ▼
service.businness.*Service         ← business rules, validation, auth checks, @Transactional
   │
   ▼
service.dao.*Dao                   ← to be implemented; EntityManager / JPA queries
   │
   ▼
service.entity.*Entity  ⇄  PostgreSQL (restaurantdb)
```

## API endpoints

All paths are served under the base path **`/api`** (`server.servlet.contextPath`). Endpoints marked 🔒 need `authorization: Bearer <access-token>`.

### Customer (`customer.json`)
| Method | Path                 | Description |
|--------|----------------------|-------------|
| POST   | `/customer/signup`   | Register a customer. Body `SignupCustomerRequest` (`first_name`*, `last_name`, `email_address`*, `contact_number`*, `password`*). Returns 201 with the customer `id`. |
| POST   | `/customer/login`    | Login with `authorization: Basic base64(contactNumber:password)`. Returns `LoginResponse` and the JWT in the `access-token` response header. |
| POST   | `/customer/logout` 🔒 | Log out and invalidate the access token. |
| PUT    | `/customer` 🔒        | Update first and last name (`UpdateCustomerRequest`). |
| PUT    | `/customer/password` 🔒 | Change password (`UpdatePasswordRequest`: old and new password). |

### Address (`address.json`)
| Method | Path                          | Description |
|--------|-------------------------------|-------------|
| POST   | `/address` 🔒                  | Save an address (`SaveAddressRequest`: flat/building, locality, city, pincode, state UUID). |
| GET    | `/address/customer` 🔒         | List the logged-in customer's saved addresses. |
| PUT    | `/address/delete/{addressId}` 🔒 | Delete an address. The tests call `DELETE /address/{address_id}` instead. |
| GET    | `/states`                     | List all states. |

### Category (`category.json`)
| Method | Path                     | Description |
|--------|--------------------------|-------------|
| GET    | `/category`              | All categories, ordered by name. |
| GET    | `/category/{categoryId}` | One category and its items. |

### Restaurant (`restaurant.json`)
| Method | Path                                 | Description |
|--------|--------------------------------------|-------------|
| GET    | `/restaurant`                        | All restaurants, ordered by rating. |
| GET    | `/restaurant/name/{restaurantName}`  | Search restaurants by name (partial match). |
| GET    | `/restaurant/category/{categoryId}`  | Restaurants in a category. |
| GET    | `/restaurant/{restaurantId}`         | Restaurant details with categories and items. |
| PUT    | `/restaurant/edit/{restaurantId}?customerRating=` 🔒 | Rate a restaurant (1–5). The running average is updated. |

### Item (`item.json`)
| Method | Path                             | Description |
|--------|----------------------------------|-------------|
| GET    | `/item/restaurant/{restaurantId}` | Top 5 most-ordered items of a restaurant. |

### Order (`order.json`)
| Method | Path                         | Description |
|--------|------------------------------|-------------|
| GET    | `/order/coupon/{couponName}` 🔒 | Look up a coupon by name. |
| GET    | `/order/customer` 🔒            | The customer's past orders, newest first. The tests call `GET /order`. |
| POST   | `/order` 🔒                     | Place an order (`SaveOrderRequest`: `address_id`*, `payment_id`*, `bill`*, `restaurant_id`*, `coupon_id`, `discount`, `item_quantities[]`). |

### Payment (`payment.json`)
| Method | Path       | Description |
|--------|------------|-------------|
| GET    | `/payment` | All payment methods. |

Once controllers exist, Swagger UI is available at `http://localhost:8080/api/swagger-ui.html`.

## Authentication

1. **Signup**: the password is hashed with `PasswordCryptographyProvider.encrypt()`. The salt and hash are stored in `CUSTOMER.salt` and `CUSTOMER.password`.
2. **Login**: the client sends `Authorization: Basic base64(contact_number:password)`. The service looks up the customer by contact number and recomputes `encrypt(password, salt)`. On a match it issues a JWT with `JwtTokenProvider` (audience = customer UUID) and stores a row in `CUSTOMER_AUTH` (`access_token`, `login_at`, `expires_at`). The token goes back in the `access-token` response header.
3. **Protected calls**: the client sends `Authorization: Bearer <token>`. The service must reject the request (`AuthorizationFailedException`) when:
   - no `CUSTOMER_AUTH` row exists for the token (`ATHR-001`),
   - `logout_at` is set (`ATHR-002`),
   - `expires_at` is in the past (`ATHR-003`).
4. **Logout**: sets `CUSTOMER_AUTH.logout_at`.

## Database schema

Defined in `FoodOrderingApp-db/src/main/resources/sql/tables.sql`. Every table has a surrogate `id SERIAL` primary key, and most also have a public `uuid` that the API exposes.

```
CUSTOMER (uuid, firstname, lastname, email, contact_number UNIQUE, password, salt)
  ├── CUSTOMER_AUTH (customer_id → CUSTOMER, access_token, login_at, logout_at, expires_at)
  ├── CUSTOMER_ADDRESS (customer_id → CUSTOMER, address_id → ADDRESS)        [M:N]
  └── ORDERS (customer_id, address_id, restaurant_id, payment_id, coupon_id, bill, discount, date)
          └── ORDER_ITEM (order_id → ORDERS, item_id → ITEM, quantity, price)

STATE (uuid, state_name)
  └── ADDRESS (flat_buil_number, locality, city, pincode, state_id → STATE, active)

RESTAURANT (restaurant_name, photo_url, customer_rating, average_price_for_two,
            number_of_customers_rated, address_id → ADDRESS)
  ├── RESTAURANT_CATEGORY (restaurant_id, category_id)                     [M:N]
  └── RESTAURANT_ITEM     (restaurant_id, item_id)                          [M:N]

CATEGORY (category_name) ── CATEGORY_ITEM (category_id, item_id)            [M:N]
ITEM (item_name, price, type)           -- type: VEG / NON_VEG
COUPON (coupon_name, percent)
PAYMENT (payment_name)
```

Notes:
- `ADDRESS.active` (default 1) supports soft delete. An address used by a past order is archived (`active = 0`) instead of being deleted.
- Most child foreign keys use `ON DELETE CASCADE`. `ORDERS → PAYMENT/RESTAURANT/ADDRESS/COUPON` do not.

## Seed data

`insert.sql` populates a demo dataset:

| Table | Rows | Notes |
|-------|------|-------|
| `CUSTOMER` | 1 | `sde@upgrad.com` / contact `1111111111` |
| `STATE` | 36 | Indian states and union territories |
| `ADDRESS` | 12 | restaurant addresses |
| `RESTAURANT` | 8 | |
| `CATEGORY` | 7 | |
| `ITEM` | 102 | |
| `RESTAURANT_ITEM` / `RESTAURANT_CATEGORY` / `CATEGORY_ITEM` | 102 / 46 / 148 | join tables |
| `COUPON` | 4 | |
| `PAYMENT` | 5 | |

## Error handling

Domain exceptions live in `service.exception`. They are checked exceptions holding a `code` and an `errorMessage`, and the controller layer is expected to turn them into the generated `ErrorResponse` (`code`, `message`) with a suitable HTTP status:

| Exception | Typical use | Expected HTTP status |
|-----------|-------------|-------------|
| `SignUpRestrictedException` (`SGR-xxx`) | duplicate contact, invalid email/contact/password, missing fields | 400 |
| `AuthenticationFailedException` (`ATH-xxx`) | bad Basic header, wrong credentials | 401 |
| `AuthorizationFailedException` (`ATHR-xxx`) | missing, logged-out or expired token; acting on another customer's resource | 403 |
| `UpdateCustomerException` (`UCR-xxx`) | invalid profile or password update | 400 |
| `SaveAddressException` (`SAR-xxx`) | empty fields, invalid pincode | 400 |
| `AddressNotFoundException` (`ANF-xxx`) | unknown address or state | 404 |
| `CategoryNotFoundException` (`CNF-xxx`) | | 404 |
| `RestaurantNotFoundException` (`RNF-xxx`) | | 404 |
| `ItemNotFoundException` (`INF-xxx`) | | 404 |
| `CouponNotFoundException` (`CPF-xxx`) | | 404 |
| `PaymentMethodNotFoundException` (`PNF-xxx`) | | 404 |
| `InvalidRatingException` (`IRE-xxx`) | rating outside 1–5 | 400 |
| `CustomerNotFoundException` | | 404 |

`UnexpectedException` together with `GenericErrorCode.GEN_001` covers unrecoverable internal errors, such as an invalid JWT secret.

## Tests

`FoodOrderingApp-api/src/test/java/com/upgrad/FoodOrderingApp/api/controller/` contains MockMvc tests for every controller (`Customer`, `Address`, `Category`, `Item`, `Order`, `Payment`, `Restaurant`). Each test boots the Spring context with `@SpringBootTest` and `@AutoConfigureMockMvc`, mocks the service layer with `@MockBean`, and checks HTTP status codes, JSON bodies and error codes.

**All test files are currently wrapped in `/* … */`**, because the services and entities they reference do not exist yet. Uncomment them as you implement each controller. They are effectively the acceptance criteria for the project.

## Configuration

`FoodOrderingApp-api/src/main/resources/application.yaml`:

```yaml
server:
  servlet:
    port: 8080          # note: Spring Boot reads `server.port`; 8080 is the default anyway
    contextPath: /api
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/restaurantdb
    username: postgres
    password: tiger1
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQL9Dialect
```

`FoodOrderingApp-db/src/main/resources/config/localhost.properties` must point at the same database. Change the credentials in both files for your environment. Do not commit real credentials.

## Getting started

Prerequisites: **JDK 8**, **Maven 3.x**, **PostgreSQL** running on `localhost:5432`. JDK 8 is recommended because Spring Boot 2.0.x and the old codegen plugin are not reliable on newer JDKs.

```bash
# 1. Create the database
psql -U postgres -c "CREATE DATABASE restaurantdb;"

# 2. Create tables and load seed data (FoodOrderingApp-db, 'setup' profile)
mvn clean install -Psetup -DskipTests

# 3. Build everything (also generates the API model classes from the Swagger specs)
mvn clean install -DskipTests

# 4. Run the API
cd FoodOrderingApp-api
mvn spring-boot:run
# → http://localhost:8080/api   (Swagger UI: /api/swagger-ui.html once controllers exist)
```

## Implementation status and known gaps

| Area | State |
|------|-------|
| Maven multi-module build, Swagger codegen | ✅ in place |
| API contracts (`endpoints/*.json`) | ✅ complete |
| DB schema and seed data | ✅ complete |
| Password hashing and JWT helpers | ✅ implemented |
| Exception classes | ✅ implemented |
| JPA entities | ❌ empty stubs; more entities needed (CustomerAuth, Category, Item, Restaurant, Order, OrderItem, Coupon, …) |
| DAOs | ❌ missing (`service.dao` package) |
| Business services | ❌ empty stubs; Category, Item, Restaurant and Order services are also missing |
| REST controllers and `@ControllerAdvice` | ❌ missing (`api.controller` package) |
| Tests | ⚠️ written but commented out |

Other things to be aware of:
- **Spec and test mismatches**: deleting an address is `PUT /address/delete/{addressId}` in the spec but `DELETE /address/{id}` in the tests. Listing orders is `GET /order/customer` in the spec but `GET /order` in the tests.
- `server.servlet.port` is not a valid Spring Boot property. Use `server.port`.
- `target/` folders, `.idea/` and `*.iml` files are committed and there is no `.gitignore`. Consider adding one.
- The DB password (`tiger1`) is hard-coded in two config files.
