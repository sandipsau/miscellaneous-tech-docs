# Kotlin + Spring WebFlux Clean Code Handbook

A pragmatic, Kotlin-first guide for building **reactive HTTP services** with **Spring WebFlux** (Reactor + coroutines), tuned for production teams.

---

## Table of Contents
1. [Design Principles in Reactive Systems](#design-principles-in-reactive-systems)
2. [Project & Language Conventions](#project--language-conventions)
3. [Controllers, Routing & DTOs](#controllers-routing--dtos)
4. [Services, Ports & Adapters](#services-ports--adapters)
5. [WebClient: Timeouts, Retries, Resilience](#webclient-timeouts-retries-resilience)
6. [Error Handling & Problem Details](#error-handling--problem-details)
7. [Validation](#validation)
8. [Logging, Tracing & Context Propagation](#logging-tracing--context-propagation)
9. [Persistence (R2DBC/Mongo) Patterns](#persistence-r2dbcmongo-patterns)
10. [Testing: Unit, Slice, Integration](#testing-unit-slice-integration)
11. [Performance, Backpressure & Tuning](#performance-backpressure--tuning)
12. [Security (Spring Security Reactive)](#security-spring-security-reactive)
13. [One-Page Checklist](#one-page-checklist)

---

## Design Principles in Reactive Systems

- **Non-blocking end-to-end:** Avoid blocking I/O on request threads. Prefer reactive clients (R2DBC, WebClient).
- **Cohesion & SRP:** One reason to change per class/handler.
- **Explicit boundaries:** Hexagonal/ports-and-adapters; isolate side effects in adapters.
- **Idempotency & timeouts:** All remote calls must have timeouts and retry/policy guards.
- **Fail fast:** Validate early; use typed results or exceptions mapped to Problem Details.
- **Prefer suspend APIs or Reactor, but don’t mix in one path:** Pick **suspend** controllers + `await*` WebClient or stick to `Mono/Flux`. Mixing increases complexity.
- **Single level of abstraction:** Keep a function either domain-centric or transport-centric, not both.

---

## Project & Language Conventions

- **Kotlin defaults:** `val` over `var`, data classes, sealed hierarchies, null-safety.
- **Module structure:** `api` (DTOs), `domain` (entities, services, ports), `adapters` (web, db, messaging).
- **Code style:** ktlint/spotless; detekt for linting; explicit visibility, `internal` when possible.
- **Configuration:** Constructor injection; externalize config via `application.yml` and `@ConfigurationProperties`.

```kotlin
@ConfigurationProperties(prefix = "downstream.orders")
data class OrdersClientProps(
    val baseUrl: String,
    val connectTimeoutMs: Int = 1000,
    val readTimeoutMs: Int = 2000
)
```

---

## Controllers, Routing & DTOs

Prefer **suspend controllers** for readability (coroutines) or functional router DSL. Keep controllers thin; delegate to services.

```kotlin
@RestController
@RequestMapping("/v1/orders")
class OrderController(private val service: OrderService) {

    @GetMapping("/{id}")
    suspend fun get(@PathVariable id: String): ResponseEntity<OrderDto> =
        service.get(id)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()

    @PostMapping
    suspend fun create(@Valid @RequestBody req: CreateOrderDto): ResponseEntity<OrderDto> {
        val created = service.create(req)
        return ResponseEntity.created(URI.create("/v1/orders/${created.id}")).body(created)
    }
}
```

**DTO tips**

- Use data classes; prefer **non-null** fields where business requires.
- Separate request/response DTOs from domain objects.
- Use `Instant`/`OffsetDateTime` in UTC; ISO-8601 on the wire.

```kotlin
data class OrderDto(
    val id: String,
    val accountId: String,
    val items: List<OrderItemDto>,
    val createdAt: Instant
)
```

---

## Services, Ports & Adapters

Define a **port** as an interface in `domain`; implement in `adapters`. Keep business logic **pure** when possible.

```kotlin
// domain
interface PricingPort { suspend fun price(cartId: String): Money }

class CheckoutService(private val pricing: PricingPort, private val clock: Clock) {
    suspend fun totals(cartId: String): Totals {
        val price = pricing.price(cartId)
        return Totals(subtotal = price, createdAt = Instant.now(clock))
    }
}
```

---

## WebClient: Timeouts, Retries, Resilience

**Build once** and inject. Configure Reactor Netty timeouts and connection pool.

```kotlin
@Configuration
class WebClientConfig {

    @Bean
    fun ordersWebClient(props: OrdersClientProps): WebClient {
        val httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, props.connectTimeoutMs)
            .responseTimeout(Duration.ofMillis(props.readTimeoutMs))
            .doOnConnected { conn ->
                conn.addHandlerLast(ReadTimeoutHandler(props.readTimeoutMs, TimeUnit.MILLISECONDS))
                conn.addHandlerLast(WriteTimeoutHandler(props.readTimeoutMs, TimeUnit.MILLISECONDS))
            }

        return WebClient.builder()
            .baseUrl(props.baseUrl)
            .clientConnector(ReactorClientHttpConnector(httpClient))
            .filter(addCorrelationIdFilter())
            .build()
    }

    private fun addCorrelationIdFilter() = ExchangeFilterFunction.ofRequestProcessor { req ->
        val cid = req.headers().firstHeader("X-Correlation-Id") ?: UUID.randomUUID().toString()
        Mono.just(ClientRequest.from(req).header("X-Correlation-Id", cid).build())
    }
}
```

**Coroutines-friendly usage:**

```kotlin
suspend fun fetchOrder(id: String): OrderDto =
    webClient.get().uri("/orders/{id}", id)
        .accept(MediaType.APPLICATION_JSON)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError) { resp ->
            resp.bodyToMono<String>().map { ClientErrorException(it) }
        }
        .onStatus(HttpStatusCode::is5xxServerError) { resp ->
            resp.bodyToMono<String>().map { ServerErrorException(it) }
        }
        .awaitBody()
```

**Retries with backoff & jitter (Resilience4j):**

```kotlin
class OrdersClient(
    private val webClient: WebClient,
    retryRegistry: RetryRegistry
) {
    private val retry = retryRegistry.retry("orders", RetryConfig.custom<Any>()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(200))
        .intervalFunction(IntervalFunction.ofExponentialBackoff(200, 2.0))
        .retryExceptions(TimeoutException::class.java, IOException::class.java)
        .build())

    suspend fun get(id: String): OrderDto =
        Retry.decorateSuspendFunction(retry) {
            webClient.get().uri("/orders/{id}", id).retrieve().awaitBody()
        }.invoke()
}
```

**Mapping empty bodies:** Use `switchIfEmpty(Mono.error(..))` in Reactor or explicit `null` handling with `suspend`.

```kotlin
// Reactor style
fun getMono(id: String): Mono<OrderDto> =
    webClient.get().uri("/orders/{id}", id)
        .retrieve()
        .bodyToMono(OrderDto::class.java)
        .switchIfEmpty(Mono.error(NotFoundException("Order $id not found")))
```

---

## Error Handling & Problem Details

Use **RFC 9457 Problem Details** for consistency.

```kotlin
data class Problem(
    val type: URI,
    val title: String,
    val status: Int,
    val detail: String? = null,
    val instance: URI? = null
)

@RestControllerAdvice
class GlobalErrors {
    @ExceptionHandler(NotFoundException::class)
    fun notFound(ex: NotFoundException, req: ServerHttpRequest): ResponseEntity<Problem> =
        ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            Problem(
                type = URI.create("https://example.com/problems/not-found"),
                title = "Not Found",
                status = 404,
                detail = ex.message,
                instance = URI.create(req.path.value())
            )
        )
}
```

- **Don’t leak internals:** sanitize messages; log the cause with structured logs.
- **One mapping place:** `@RestControllerAdvice` converts domain/infra exceptions to HTTP.

---

## Validation

- Use **Jakarta Bean Validation** on DTOs; re-validate at domain boundaries if critical.
- Prefer **value classes** to encode invariants (`Email`, `Money`).

```kotlin
data class CreateOrderDto(
    @field:NotBlank val accountId: String,
    @field:Size(min = 1) val items: List<OrderItemDto>
)
```

---

## Logging, Tracing & Context Propagation

- **Structured logs** (JSON) with MDC keys: `traceId`, `spanId`, `correlationId`, `accountId`.
- With coroutines + Reactor, enable **context bridging** (Spring Boot 3+ ships Micrometer context propagation). When rolling your own:
  - Add correlation ID in a filter.
  - Use `kotlinx-coroutines-slf4j` `MDCContext()` for coroutine blocks where needed.
- **Do not log PII/secrets**; mask tokens.

```kotlin
withContext(MDCContext(mapOf("correlationId" to correlationId))) {
    logger.info { "pricing_requested cartId=$cartId" }
}
```

---

## Persistence (R2DBC/Mongo) Patterns

- Use reactive drivers (R2DBC, reactive Mongo).
- Return `suspend` or Reactor types, not blocking `ResultSet`/`JdbcTemplate`.
- **Repository methods** should be small, focused; keep joins in DB or projection queries.

```kotlin
interface OrderRepository {
    suspend fun findById(id: String): Order?
    suspend fun upsert(order: Order): Order
}
```

---

## Testing: Unit, Slice, Integration

**1) Unit w/ coroutines:**

```kotlin
class CheckoutServiceTest {

    private val pricing = mockk<PricingPort>()
    private val clock = Clock.fixed(Instant.parse("2025-08-20T00:00:00Z"), ZoneOffset.UTC)
    private val service = CheckoutService(pricing, clock)

    @Test
    fun `totals uses pricing and clock`() = runTest {
        coEvery { pricing.price("C1") } returns Money(1234)
        val totals = service.totals("C1")
        assertEquals(Instant.parse("2025-08-20T00:00:00Z"), totals.createdAt)
        assertEquals(Money(1234), totals.subtotal)
    }
}
```

**2) WebTestClient (slice):**

```kotlin
@WebFluxTest(controllers = [OrderController::class])
@Import(OrderService::class)
class OrderControllerTest(@Autowired val client: WebTestClient) {

    @MockkBean lateinit var service: OrderService

    @Test
    fun `GET 200 with body`() {
        every { runBlocking { service.get("O1") } } returns OrderDto("O1", "A1", emptyList(), Instant.EPOCH)

        client.get().uri("/v1/orders/O1")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.id").isEqualTo("O1")
    }
}
```

**3) Mocking WebClient without network:** stub the `ExchangeFunction`.

```kotlin
class WebClientStub(private val resp: ClientResponse) : ExchangeFunction {
    override fun exchange(request: ClientRequest): Mono<ClientResponse> = Mono.just(resp)
}

// usage
val response = ClientResponse.create(HttpStatus.OK)
    .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .body("{\"id\":\"O1\",\"accountId\":\"A1\",\"items\":[],\"createdAt\":\"1970-01-01T00:00:00Z\"}")
    .build()

val client = WebClient.builder().exchangeFunction(WebClientStub(response)).build()
```

**4) StepVerifier for Reactor code:** Prefer this over `runBlocking` for `Mono/Flux` paths.

```kotlin
StepVerifier.create(service.getMono("O1"))
    .expectNextMatches { it.id == "O1" }
    .verifyComplete()
```

**5) Integration:** Use **WireMock** or **MockWebServer** to simulate HTTP APIs; use Testcontainers for DBs; run with `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `WebTestClient`.

---

## Performance, Backpressure & Tuning

- **Backpressure-aware**: prefer streaming (`Flux`) for large responses; paginate otherwise.
- **Tune Netty:** connection pool size, timeouts, and `maxInMemorySize` for codecs when handling large payloads.
- **Avoid `collectList()` on unbounded streams.**
- **Cache carefully:** TTLs and size bounds; document invalidation.
- **Metrics:** request latencies, percentiles, retries, timeouts; include downstream call metrics.

---

## Security (Spring Security Reactive)

- Use the **reactive filter chain**; avoid blocking user stores.
- Method-level security with `@PreAuthorize` is supported in reactive.
- Propagate user identity (subject) through context; avoid static security contexts.
- Sanitize error responses; no stack traces to clients.

```kotlin
@EnableWebFluxSecurity
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain =
        http
            .csrf { it.disable() }
            .authorizeExchange { ex ->
                ex.pathMatchers(HttpMethod.GET, "/v1/health").permitAll()
                  .anyExchange().authenticated()
            }
            .oauth2ResourceServer { it.jwt() }
            .build()
}
```

---

## One-Page Checklist

- [ ] Controllers are **suspend** or pure Reactor (not mixed)  
- [ ] No blocking calls on event loops (`runBlocking`, JDBC, file I/O)  
- [ ] WebClient configured with **timeouts**, **connection pool**, correlation ID filter  
- [ ] Retries are bounded with **exponential backoff + jitter**; idempotency ensured  
- [ ] Global `@RestControllerAdvice` maps exceptions to **Problem Details**  
- [ ] DTOs validated; domain invariants enforced with value classes  
- [ ] Logs are structured; **no PII/secrets**; correlation IDs present; context propagated  
- [ ] Repositories use **R2DBC/Mongo reactive**; no `collectList()` on big streams  
- [ ] Tests: unit (runTest), WebTestClient slice, WireMock/Testcontainers integration  
- [ ] Metrics & tracing in place; high-cardinality labels avoided  
- [ ] Security via reactive chain; least-privilege scopes/roles  
- [ ] Feature flags and rollout plan; graceful shutdown and timeouts verified

---

**Tip:** Keep controllers dumb, services smart, adapters tiny. Timeouts everywhere, retries rarely, metrics always.
