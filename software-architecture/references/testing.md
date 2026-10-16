# Testing Strategy

Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). Test frameworks per language are in [language-adaptation.md](language-adaptation.md#ecosystem-toolbox). The layered architecture exists largely to make this strategy cheap: the domain and use cases are testable without any infrastructure.

## Table of Contents

1. [Test Portfolio](#test-portfolio)
2. [Unit Testing](#unit-testing)
3. [Test Doubles: Fakes over Mocks](#test-doubles-fakes-over-mocks)
4. [Integration Testing](#integration-testing)
5. [Contract Testing](#contract-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [Test Data Builders and Fixtures](#test-data-builders-and-fixtures)
8. [Other Test Types](#other-test-types)
9. [Testing Practices](#testing-practices)

---

## Test Portfolio

| Layer | What | Dependencies | Speed | Share |
|---|---|---|---|---|
| **Unit** | Domain entities, value objects, domain services, use cases | None (fakes for ports) | ms | Most tests |
| **Integration** | Repository adapters, cache/broker adapters, HTTP handlers with real framework | Real DB/cache/broker in containers | 100s of ms | Every adapter |
| **Contract** | API/event compatibility between services | Contract files / broker | fast | Every service boundary |
| **End-to-end** | Critical user journeys through the deployed system | Everything | seconds | A handful |

Test **behaviour through public interfaces**, not implementation details — tests should survive refactoring.

---

## Unit Testing

### Domain logic — no doubles needed

```
describe "Order" {
  test "create computes subtotal" {
    order = Order.create(customerId = aCustomerId(), items = [anItem(quantity = 2, unitPrice = eur("10.00"))], clock = fixedClock)
    expect(order.status).toEqual(PENDING)
    expect(order.subtotal()).toEqual(eur("20.00"))
  }

  test "create without items is rejected" {
    error = expectFailure(() -> Order.create(aCustomerId(), items = [], clock = fixedClock))
    expect(error).toBe(RuleViolated)
    expect(error.rule).toEqual("order_must_have_items")
  }

  test "confirm emits OrderConfirmed" {
    order = anOrder().build()
    order.confirm(fixedClock)
    expect(order.status).toEqual(CONFIRMED)
    expect(order.pullEvents()).toContainOnly(OrderConfirmed(orderId = order.id, ...))
  }

  test "confirming twice is rejected" {
    order = anOrder().confirmed().build()
    expect(() -> order.confirm(fixedClock)).toFailWith(RuleViolated)
  }
}

describe "Email" {
  test "accepts and normalises a valid address" { expect(Email(" Ada@Example.com ").value).toEqual("ada@example.com") }
  test "rejects an invalid address"            { expect(() -> Email("not-an-email")).toFailWith(InvalidInput) }
}
```

### Use cases — with fakes for ports

```
describe "GetUserUseCase" {
  setup {
    users = InMemoryUserRepository()
    useCase = GetUserUseCase(users)
  }

  test "returns the user" {
    users.save(aUser(id = "u1", email = "ada@example.com"))
    expect(useCase.execute("u1").email).toEqual("ada@example.com")
  }

  test "fails with NotFound for unknown id" {
    expect(() -> useCase.execute("missing")).toFailWith(NotFound)
  }
}

describe "CachedUserRepository" {
  test "serves from cache without hitting the inner repository" {
    inner = CountingUserRepository(InMemoryUserRepository(with = [aUser(id = "u1")]))
    repo = CachedUserRepository(inner, InMemoryCache(), ttl = 5 minutes)
    repo.getById("u1"); repo.getById("u1")
    expect(inner.getByIdCalls).toEqual(1)
  }
}
```

Use parameterised/table-driven tests for many input variations (JUnit `@ParameterizedTest`, xUnit `[Theory]`, `test.each`, `pytest.mark.parametrize`, Go table-driven tests with `t.Run`, `rstest`, Pest datasets, RSpec shared examples, ExUnit `for` comprehensions).

---

## Test Doubles: Fakes over Mocks

| Double | What | Use when |
|---|---|---|
| **Fake** | A working lightweight implementation (in-memory repository, in-memory event bus, fixed clock) | Default for ports you own — tests read naturally and don't break on refactors |
| **Stub** | Returns canned answers | Simulating a specific response/failure from a dependency |
| **Mock / spy** | Records calls for verification | The interaction itself *is* the behaviour (an email was sent, an event was published) |

```
class InMemoryUserRepository implements UserRepository {
  private data: Map<UserId, User>
  getById(id) { return data.get(id) }
  save(user)  { data.put(user.id, user) }
}

class FixedClock(instant) implements Clock { now() { return instant } }
```

Over-mocking (asserting every call, mocking types you don't own) produces brittle tests that verify the implementation rather than the behaviour. Don't mock the database driver — use an integration test instead. **Run the same contract test suite against the fake and the real adapter** to keep them honest.

---

## Integration Testing

Test adapters against **real** infrastructure, started in containers. Testcontainers has libraries for Java, .NET, Node, Python, Go, Rust, PHP, Ruby, Elixir (or use `docker compose` in CI).

```
suite "SqlUserRepository" {
  beforeAll {
    postgres = Container("postgres:16").start()
    pool = Database.connect(postgres.connectionUrl)
    runMigrations(pool)                                  // the real migrations, not a hand-written schema
  }
  afterAll { pool.close(); postgres.stop() }

  beforeEach { conn = pool.acquire(); conn.begin(); repo = SqlUserRepository(conn) }
  afterEach  { conn.rollback(); pool.release(conn) }     // isolation: every test rolls back

  test "save then get round-trips" {
    user = aUser(email = "ada@example.com")
    repo.save(user)
    expect(repo.getById(user.id)).toEqual(user)
  }

  test "get by email is case-insensitive" {
    repo.save(aUser(email = "Ada@Example.com"))
    expect(repo.getByEmail(Email("ada@example.com"))).toBePresent()
  }

  test "loading orders with items uses bounded queries" {
    seedOrdersWithItems(count = 20)
    queries = countQueries { orderRepo.findByCustomer(customerId, page(20)) }
    expect(queries).toBeLessThanOrEqual(2)               // guards against N+1 regressions
  }
}
```

**HTTP layer**: start the app in-process with its real router, middleware, and serialisation, but with fakes or containers behind it (Spring `MockMvc`/`WebTestClient`, ASP.NET `WebApplicationFactory`, `supertest`/`fastify.inject`, FastAPI/Starlette `TestClient`, Go `httptest`, Axum `oneshot`, Laravel HTTP tests, Rails request specs, Phoenix `ConnTest`).

```
suite "Users API" {
  test "POST /v1/users creates a user" {
    response = client.post("/v1/users", json = { email: "new@example.com", name: "New User", password: "correct-horse-battery" })
    expect(response.status).toEqual(201)
    expect(response.json.email).toEqual("new@example.com")
    expect(response.headers.Location).toMatch("/v1/users/")
  }
  test "GET /v1/users/{id} requires authentication" {
    expect(client.get("/v1/users/123").status).toEqual(401)
  }
  test "validation errors use problem+json" {
    response = client.post("/v1/users", json = { email: "nope" })
    expect(response.status).toEqual(422)
    expect(response.headers["Content-Type"]).toContain("application/problem+json")
  }
}
```

For **third-party HTTP APIs**, use a local HTTP stub server (WireMock, MockServer, `nock`/`msw`, `respx`, `httptest.Server`, `wiremock-rs`) to simulate success, errors, slowness, and timeouts.

---

## Contract Testing

Between services, end-to-end tests are slow and flaky. Instead:

- **Consumer-driven contracts** (Pact, Spring Cloud Contract): each consumer publishes the requests/responses it relies on; the provider verifies them in its CI.
- **Schema checks**: lint OpenAPI/Protobuf/AsyncAPI and fail CI on breaking changes (`buf breaking`, `oasdiff`, schema registry compatibility rules).
- **Event contracts**: producers test that emitted events validate against the published schema; consumers test they can parse fixtures of every supported version.

---

## End-to-End Testing

- Cover only **critical journeys** (sign up → place order → pay → receive confirmation).
- Run against a production-like environment with seeded data; make tests independent and idempotent.
- Browser/app UIs: Playwright, Cypress, Selenium; mobile: XCUITest, Espresso, Maestro, Detox, Flutter integration tests.
- Track flakiness; quarantine and fix flaky tests rather than retrying them forever.

---

## Test Data Builders and Fixtures

Builders give each test valid defaults and let it state only what matters:

```
anOrder() = OrderBuilder(
  customerId = aCustomerId(),
  items = [anItem()],
  status = PENDING)

class OrderBuilder {
  withItems(items) -> OrderBuilder { ... }
  confirmed() -> OrderBuilder { status = CONFIRMED; return this }
  build() -> Order { ... uses domain factory so invariants hold ... }
}

// Persistence factories for integration tests (unique values per call)
userFactory.create(email = sequence(n -> "user" + n + "@example.com"))
orderFactory.createMany(3, customer = user)
```

Ecosystem tools: Instancio/EasyRandom (Java), AutoFixture/Bogus (.NET), `fishery`/`@faker-js/faker` (TS), `factory_boy`/`polyfactory` (Python), `gofakeit` (Go), `fake` (Rust), Laravel model factories, FactoryBot (Ruby), ExMachina (Elixir).

- Prefer builders over large shared fixture files — shared fixtures couple unrelated tests.
- Use **fixed clocks and seeded randomness** for determinism.

---

## Other Test Types

- **Property-based tests** for invariants over many generated inputs (jqwik, FsCheck, fast-check, Hypothesis, `gopter`/`rapid`, `proptest`, StreamData): e.g. `Money.plus` is associative; serialising then deserialising returns the original.
- **Performance/load tests** (k6, Gatling, Locust, JMeter, vegeta) for critical endpoints, run before major releases and against SLOs.
- **Architecture tests** enforcing layer rules (ArchUnit, NetArchTest, dependency-cruiser, `import-linter`, Deptrac).
- **Mutation testing** (PIT, Stryker, mutmut, cargo-mutants) to check the suite actually catches bugs in core logic.
- **Security tests**: dependency scanning, SAST, DAST (OWASP ZAP) — see [security.md](security.md#supply-chain).

---

## Testing Practices

- **Arrange–Act–Assert** structure; one behaviour per test; descriptive names (`confirming_twice_is_rejected`).
- Tests run in **CI on every change**; the default branch is always green.
- Measure coverage to find untested areas, but don't chase a number — 100% coverage of getters is worthless; untested pricing rules are dangerous.
- Fast feedback: unit tests in seconds locally; integration in minutes in CI; parallelise where isolation allows.
- Treat test code as production code: readable, refactored, reviewed.
