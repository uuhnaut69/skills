# Spring Boot 4 Features

## Jackson 3 Migration

Spring Boot 4 uses Jackson 3 with the new `tools.jackson` package namespace.

### JsonMapper (replaces ObjectMapper)
```java
import tools.jackson.databind.JsonMapper;

@Configuration
public class JacksonConfig {

    @Bean
    public JsonMapper jsonMapper() {
        return JsonMapper.builder()
            .findAndAddModules()
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .enable(SerializationFeature.INDENT_OUTPUT)
            .build();
    }
}
```

### Key Changes from Jackson 2
- Package: `com.fasterxml.jackson` → `tools.jackson`
- Main class: `ObjectMapper` → `JsonMapper`
- Builder pattern is now the standard way to configure
- Immutable configuration by default

### Module Registration
```java
JsonMapper mapper = JsonMapper.builder()
    .addModule(new JavaTimeModule())
    .addModule(new ParameterNamesModule())
    .findAndAddModules() // Auto-discover modules on classpath
    .build();
```

### Exception Handling Changes
```java
// Jackson 2 (old) - checked exception
try {
    String json = objectMapper.writeValueAsString(order);
} catch (JsonProcessingException e) {  // Checked exception
    throw new RuntimeException(e);
}

// Jackson 3 (new) - unchecked exception
String json = jsonMapper.writeValueAsString(order);  // JacksonException is unchecked
```

### Common Annotations Migration
Annotations move to new package but names stay the same:
```java
// Jackson 2
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonFormat;

// Jackson 3
import tools.jackson.annotation.JsonProperty;
import tools.jackson.annotation.JsonIgnore;
import tools.jackson.annotation.JsonFormat;
```

## JSpecify Null Safety

Spring Boot 4 adopts JSpecify annotations for null safety.

### Package-Level Null Marking
```java
// package-info.java
@NullMarked
package com.example.orders;

import org.jspecify.annotations.NullMarked;
```

### Nullable Parameters and Returns
```java
import org.jspecify.annotations.Nullable;

@Service
@NullMarked
public class OrderService {

    // Returns null when not found (explicit)
    public @Nullable Order findById(String id) {
        return orderRepository.findById(id).orElse(null);
    }

    // Parameters are non-null by default in @NullMarked package
    public Order create(CreateOrderRequest request) {
        // request is guaranteed non-null
        return orderRepository.save(mapToEntity(request));
    }

    // Explicitly nullable parameter
    public List<Order> search(@Nullable String status) {
        if (status == null) {
            return orderRepository.findAll();
        }
        return orderRepository.findByStatus(status);
    }
}
```

### Benefits
- Compile-time null checking with compatible tools
- Clear API contracts
- Works with Kotlin interop
- IDE support for null warnings

## RestTestClient

Unified testing API replacing MockMvc and WebTestClient distinctions.

### Binding Methods
RestTestClient provides five binding strategies for different test scenarios:

| Method | Use Case |
|--------|----------|
| `bindToController(controller)` | Unit test controllers in isolation with mocks |
| `bindToRouterFunction(routerFunction)` | Test functional endpoints |
| `bindToMockMvc(mockMvc)` | Integration with Spring MVC features (validation, security) |
| `bindToApplicationContext(context)` | Full application context without server |
| `bindToServer(uri)` | End-to-end tests against running server |

### Unit Test with bindToController
```java
class OrderControllerUnitTest {

    private final OrderService orderService = mock(OrderService.class);
    private final RestTestClient client = RestTestClient
        .bindToController(new OrderController(orderService))
        .build();

    @Test
    void shouldGetOrder() {
        when(orderService.findById("123")).thenReturn(new Order("123", "PENDING"));

        client.get()
            .uri("/api/orders/123")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.id").isEqualTo("123");
    }
}
```

### Integration Test with bindToMockMvc
```java
@WebMvcTest(OrderController.class)
class OrderControllerIntegrationTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    OrderService orderService;

    @Test
    void shouldValidateRequest() {
        RestTestClient client = RestTestClient.bindToMockMvc(mockMvc).build();

        client.post()
            .uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .body(new CreateOrderRequest("", List.of()))
            .exchange()
            .expectStatus().isBadRequest();
    }
}
```

### End-to-End Test with bindToServer
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderApiE2ETest {

    @LocalServerPort
    int port;

    @Test
    void shouldCreateAndRetrieveOrder() {
        RestTestClient client = RestTestClient
            .bindToServer(URI.create("http://localhost:" + port))
            .build();

        // Create order
        var response = client.post()
            .uri("/api/orders")
            .body(new CreateOrderRequest("cust-1", List.of(item)))
            .exchange()
            .expectStatus().isCreated()
            .expectBody(OrderResponse.class)
            .returnResult()
            .getResponseBody();

        // Retrieve order
        client.get()
            .uri("/api/orders/" + response.id())
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.status").isEqualTo("PENDING");
    }
}
```

### Common Assertions
```java
client.get()
    .uri("/api/orders")
    .exchange()
    // Status assertions
    .expectStatus().isOk()
    .expectStatus().is2xxSuccessful()
    // Header assertions
    .expectHeader().contentType(MediaType.APPLICATION_JSON)
    .expectHeader().exists("X-Request-Id")
    // Body assertions with JSONPath
    .expectBody()
    .jsonPath("$.length()").isEqualTo(10)
    .jsonPath("$[0].id").isNotEmpty()
    .json(expectedJson);
```

### Basic Usage
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class OrderApiTests {

    @Autowired
    RestTestClient restTestClient;

    @Test
    void shouldCreateOrder() {
        var request = new CreateOrderRequest("customer-1", List.of(item));

        restTestClient.post()
            .uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(OrderResponse.class)
            .value(order -> {
                assertThat(order.id()).isNotNull();
                assertThat(order.status()).isEqualTo("PENDING");
            });
    }

    @Test
    void shouldReturnNotFoundForMissingOrder() {
        restTestClient.get()
            .uri("/api/orders/nonexistent")
            .exchange()
            .expectStatus().isNotFound()
            .expectBody(ProblemDetail.class)
            .value(problem -> {
                assertThat(problem.getTitle()).isEqualTo("Order Not Found");
            });
    }
}
```

### Slice Test Integration
```java
@WebMvcTest(OrderController.class)
class OrderControllerTests {

    @Autowired
    RestTestClient restTestClient;

    @MockitoBean
    OrderService orderService;

    @Test
    void shouldValidateRequest() {
        var invalidRequest = new CreateOrderRequest("", List.of());

        restTestClient.post()
            .uri("/api/orders")
            .body(invalidRequest)
            .exchange()
            .expectStatus().isBadRequest();
    }
}
```

## Built-in Resilience

Spring Boot 4 includes native resilience patterns without external libraries.

### Enable Resilient Methods
```java
@SpringBootApplication
@EnableResilientMethods  // Required to activate @Retryable and @ConcurrencyLimit
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Retry with @Retryable
```java
@Service
public class PaymentService {

    @Retryable(
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, jitter = 0.1),
        retryFor = {PaymentGatewayException.class, TimeoutException.class}
    )
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentGateway.charge(request);
    }

    @Recover
    public PaymentResult recoverPayment(PaymentGatewayException ex, PaymentRequest request) {
        log.error("Payment failed after retries: {}", ex.getMessage());
        return PaymentResult.failed(request.orderId(), ex.getMessage());
    }
}
```

### @Retryable Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `maxAttempts` | Maximum retry attempts | 3 |
| `delay` | Initial delay in milliseconds | 1000 |
| `multiplier` | Backoff multiplier for exponential retry | 2.0 |
| `maxDelay` | Maximum delay between retries | 30000 |
| `jitter` | Random jitter factor (0.0 to 1.0) | 0.0 |
| `retryFor` | Exceptions that trigger retry | All |
| `noRetryFor` | Exceptions that skip retry | None |
```

### Concurrency Limiting
```java
@Service
public class ReportService {

    @ConcurrencyLimit(permits = 5) // Max 5 concurrent executions
    public Report generateReport(ReportRequest request) {
        // CPU-intensive report generation
        return reportGenerator.generate(request);
    }
}
```

### Configuration
```yaml
spring:
  resilience:
    retry:
      default:
        max-attempts: 3
        backoff:
          initial-interval: 1s
          multiplier: 2
          max-interval: 30s
    concurrency:
      default:
        permits: 10
```

### Combining Annotations
```java
@Service
public class ExternalApiService {

    @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 500, multiplier = 2))
    @ConcurrencyLimit(permits = 10)  // Limit concurrent calls while retrying
    public ApiResponse callExternalService(ApiRequest request) {
        return externalClient.call(request);
    }
}
```

## Modular Auto-Configuration

Spring Boot 4 enables selective auto-configuration loading. This is a **breaking change** from previous versions.

### Why Modular?
- Smaller application footprint
- Faster startup times
- Clearer dependency management
- Only load what you need

### Quick Migration (Classic Module)
For immediate migration, add the classic module to restore Spring Boot 3.x behavior:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-classic</artifactId>
</dependency>
```

### Modular Approach (Recommended)
Add only the modules you need:
```xml
<!-- Web MVC -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-web</artifactId>
</dependency>

<!-- JPA + Hibernate -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-data-jpa</artifactId>
</dependency>

<!-- Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-security</artifactId>
</dependency>

<!-- REST Client -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-restclient</artifactId>
</dependency>
```

### Common Modules
| Module | Description |
|--------|-------------|
| `spring-boot-autoconfigure-web` | Spring MVC, embedded server |
| `spring-boot-autoconfigure-data-jpa` | JPA repositories, Hibernate |
| `spring-boot-autoconfigure-data-jdbc` | JDBC repositories |
| `spring-boot-autoconfigure-security` | Spring Security |
| `spring-boot-autoconfigure-restclient` | RestClient, HTTP Interface Clients |
| `spring-boot-autoconfigure-validation` | Bean Validation |
| `spring-boot-autoconfigure-cache` | Caching abstraction |
| `spring-boot-autoconfigure-actuator` | Actuator endpoints |

### Migration Strategy
1. **Immediate**: Add `spring-boot-autoconfigure-classic` for full compatibility
2. **Gradual**: Replace classic with specific modules one at a time
3. **Optimal**: Start new projects with only required modules

### Opt-in Configuration
```java
@SpringBootApplication(
    excludeAutoConfiguration = {
        DataSourceAutoConfiguration.class,
        SecurityAutoConfiguration.class
    }
)
@ImportAutoConfiguration({
    WebMvcAutoConfiguration.class,
    JacksonAutoConfiguration.class
})
public class LightweightApplication {
    public static void main(String[] args) {
        SpringApplication.run(LightweightApplication.class, args);
    }
}
```

### Module Groups
```yaml
spring:
  autoconfigure:
    modules:
      - web          # Web MVC essentials
      - data-jpa     # JPA + Hibernate
      - security     # Spring Security
      - observability # Micrometer + tracing
```

### Custom Module Definition
```java
@AutoConfigurationModule(
    name = "orders",
    requires = {"web", "data-jpa"},
    provides = {OrderAutoConfiguration.class, OrderRepositoryConfiguration.class}
)
public class OrderModule {
}
```

## Virtual Threads (Project Loom)

Spring Boot 4 has first-class virtual thread support.

### Enable Virtual Threads
```yaml
spring:
  threads:
    virtual:
      enabled: true
```

### Per-Component Configuration
```java
@RestController
@RequestMapping("/api/orders")
@VirtualThreads // Use virtual threads for all handlers
public class OrderController {

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        // Runs on virtual thread
        return orderService.findById(id);
    }
}
```

### Blocking Operations
Virtual threads excel with blocking I/O:
```java
@Service
public class OrderAggregator {

    public AggregatedOrder aggregate(String orderId) {
        // All these blocking calls run efficiently on virtual threads
        var order = orderService.findById(orderId);
        var customer = customerService.findById(order.customerId());
        var payments = paymentService.findByOrderId(orderId);
        var shipments = shipmentService.findByOrderId(orderId);

        return new AggregatedOrder(order, customer, payments, shipments);
    }
}
```

## HTTP Interface Clients (Spring Boot 4)

Spring Boot 4 simplifies HTTP Interface Client registration with `@ImportHttpServices`.

### Define the Interface
```java
@HttpExchange("/api/payments")
public interface PaymentClient {

    @PostExchange
    PaymentResponse createPayment(@RequestBody PaymentRequest request);

    @GetExchange("/{id}")
    PaymentResponse getPayment(@PathVariable String id);

    @GetExchange
    List<PaymentResponse> listPayments(@RequestParam String orderId);
}
```

### Register with @ImportHttpServices (Spring Boot 4)
```java
@SpringBootApplication
@ImportHttpServices(clients = {PaymentClient.class, InventoryClient.class})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Configuration in application.yml:
```yaml
spring:
  http:
    services:
      payment-client:
        base-url: https://api.payments.example.com
        connect-timeout: 5s
        read-timeout: 30s
      inventory-client:
        base-url: https://api.inventory.example.com
```

### Old Way (Spring Boot 3.x) - Manual Factory Setup
```java
@Configuration
public class HttpClientsConfig {

    @Bean
    public PaymentClient paymentClient(RestClient.Builder builder) {
        RestClient restClient = builder
            .baseUrl("https://api.payments.example.com")
            .build();

        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(PaymentClient.class);
    }

    @Bean
    public InventoryClient inventoryClient(RestClient.Builder builder) {
        // Repeat boilerplate for each client...
    }
}
```

### Common HTTP Exchange Annotations
| Annotation | HTTP Method |
|------------|-------------|
| `@HttpExchange` | Class-level base path, or any method |
| `@GetExchange` | GET |
| `@PostExchange` | POST |
| `@PutExchange` | PUT |
| `@PatchExchange` | PATCH |
| `@DeleteExchange` | DELETE |
