# Testing Spring Applications

## Dependencies

```xml
<dependencies>
    <!-- Spring Boot Test Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- For async testing -->
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Testcontainers Setup

### Shared Container with Abstract Base Class
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class AbstractIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer(DockerImageName.parse("redis:7-alpine"));
}

// Extend in tests
class OrderIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        var request = new CreateOrderRequest("customer-1", List.of(item));
        var response = restTemplate.postForEntity("/api/orders", request, OrderResponse.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

### @ServiceConnection (Spring Boot 3.1+)
```java
// Automatically configures datasource from container
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

// No need for @DynamicPropertySource when using @ServiceConnection
```

### Manual Property Registration (Pre-3.1)
```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

@DynamicPropertySource
static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
}
```

### Common Containers
```java
// PostgreSQL
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

// MySQL
@Container
@ServiceConnection
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8");

// Redis
@Container
@ServiceConnection
static RedisContainer redis = new RedisContainer(DockerImageName.parse("redis:7-alpine"));

// Kafka
@Container
@ServiceConnection
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

// MongoDB
@Container
@ServiceConnection
static MongoDBContainer mongo = new MongoDBContainer("mongo:7");

// Elasticsearch
@Container
@ServiceConnection
static ElasticsearchContainer elastic = new ElasticsearchContainer("elasticsearch:8.11.0");
```

## Container Reuse

Speed up local development by reusing containers across test runs.

### Enable Reuse
```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withReuse(true);
```

### Configure Testcontainers
Create `~/.testcontainers.properties`:
```properties
testcontainers.reuse.enable=true
```

### Disable Ryuk for Reuse
Ryuk cleans up containers after tests. Disable for reuse:
```properties
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
ryuk.disabled=true
```

## Test Slices

### @WebMvcTest - Controller Layer
Tests controllers with MockMvc, no server started.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockitoBean
    private OrderService orderService;

    @Test
    void createOrder_ValidRequest_ReturnsCreated() throws Exception {
        var request = new CreateOrderRequest("cust-1", List.of(item));
        var response = new OrderResponse("ord-1", "cust-1", "PENDING", List.of(), BigDecimal.TEN, Instant.now());

        when(orderService.create(any())).thenReturn(response);

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("ord-1"))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    void createOrder_InvalidRequest_ReturnsBadRequest() throws Exception {
        var invalidRequest = new CreateOrderRequest("", List.of());

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.title").value("Validation Error"));
    }

    @Test
    void getOrder_NotFound_ReturnsNotFound() throws Exception {
        when(orderService.findById("123")).thenThrow(new OrderNotFoundException("123"));

        mockMvc.perform(get("/api/orders/123"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.title").value("Order Not Found"));
    }
}
```

### @DataJpaTest - Repository Layer
Tests JPA repositories with an embedded database.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE) // Use real DB with Testcontainers
@Testcontainers
class OrderRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void findByCustomerId_ReturnsMatchingOrders() {
        // Given
        Order order1 = Order.create("cust-1");
        Order order2 = Order.create("cust-1");
        Order order3 = Order.create("cust-2");

        entityManager.persist(order1);
        entityManager.persist(order2);
        entityManager.persist(order3);
        entityManager.flush();

        // When
        List<Order> results = orderRepository.findByCustomerId("cust-1");

        // Then
        assertThat(results).hasSize(2);
        assertThat(results).allMatch(o -> o.getCustomerId().equals("cust-1"));
    }

    @Test
    void findByIdWithLines_ReturnsFetchedAssociation() {
        Order order = Order.create("cust-1");
        order.addLine("prod-1", 2, BigDecimal.TEN);
        entityManager.persist(order);
        entityManager.flush();
        entityManager.clear();

        Optional<Order> found = orderRepository.findByIdWithLines(order.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getLines()).hasSize(1);
    }
}
```

### @SpringBootTest - Full Integration
Tests the complete application with a running server.

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderApiIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void fullOrderLifecycle() {
        // Create
        var createRequest = new CreateOrderRequest("cust-1", List.of(item));
        var createResponse = restTemplate.postForEntity("/api/orders", createRequest, OrderResponse.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderId = createResponse.getBody().id();

        // Get
        var getResponse = restTemplate.getForEntity("/api/orders/" + orderId, OrderResponse.class);
        assertThat(getResponse.getBody().status()).isEqualTo("PENDING");

        // Confirm
        var confirmResponse = restTemplate.postForEntity(
            "/api/orders/" + orderId + "/confirm", null, OrderResponse.class);
        assertThat(confirmResponse.getBody().status()).isEqualTo("CONFIRMED");
    }
}
```

### Test Slice Reference

| Slice | What It Tests | What's Loaded |
|-------|---------------|---------------|
| `@WebMvcTest` | Controllers | Web layer, MockMvc, no server |
| `@WebFluxTest` | WebFlux controllers | WebFlux, WebTestClient |
| `@DataJpaTest` | JPA repositories | JPA, embedded DB by default |
| `@DataJdbcTest` | JDBC repositories | JDBC, embedded DB |
| `@DataMongoTest` | MongoDB repositories | MongoDB |
| `@DataRedisTest` | Redis repositories | Redis |
| `@RestClientTest` | REST clients | RestClient, MockRestServiceServer |
| `@JsonTest` | JSON serialization | Jackson ObjectMapper |
| `@SpringBootTest` | Full application | Everything |

**NOTE:** For Spring Boot 4 RestTestClient (unified testing API), see `spring-boot-4.md`.

## Test Data

### @Sql for Script Loading
```java
@SpringBootTest
@Sql(scripts = "/test-data/orders.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/test-data/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class OrderServiceTest {

    @Test
    void shouldFindExistingOrders() {
        List<Order> orders = orderService.findByCustomerId("test-customer");
        assertThat(orders).hasSize(3);
    }
}
```

### Test Data Builders
```java
public class TestOrderBuilder {
    private String id = UUID.randomUUID().toString();
    private String customerId = "test-customer";
    private OrderStatus status = OrderStatus.PENDING;
    private List<OrderLine> lines = new ArrayList<>();

    public static TestOrderBuilder anOrder() {
        return new TestOrderBuilder();
    }

    public TestOrderBuilder withCustomerId(String customerId) {
        this.customerId = customerId;
        return this;
    }

    public TestOrderBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }

    public TestOrderBuilder withLine(String productId, int quantity) {
        this.lines.add(new OrderLine(productId, quantity, BigDecimal.TEN));
        return this;
    }

    public Order build() {
        Order order = Order.create(customerId);
        lines.forEach(line -> order.addLine(line.getProductId(), line.getQuantity(), line.getUnitPrice()));
        return order;
    }
}

// Usage
@Test
void shouldCalculateTotal() {
    Order order = anOrder()
        .withCustomerId("cust-1")
        .withLine("prod-1", 2)
        .withLine("prod-2", 3)
        .build();

    assertThat(order.getTotal()).isEqualTo(new BigDecimal("50.00"));
}
```

### Test Fixtures
```java
public class TestFixtures {

    public static CreateOrderRequest validCreateRequest() {
        return new CreateOrderRequest(
            "customer-123",
            List.of(new OrderItemRequest("product-1", 2))
        );
    }

    public static Order pendingOrder() {
        Order order = Order.create("customer-123");
        order.addLine("product-1", 2, BigDecimal.TEN);
        return order;
    }

    public static Order confirmedOrder() {
        Order order = pendingOrder();
        order.confirm();
        return order;
    }
}
```

## Async Testing

### With Awaitility
```java
@SpringBootTest
class OrderEventTest {

    @Autowired
    private OrderService orderService;

    @SpyBean
    private NotificationService notificationService;

    @Test
    void shouldSendNotificationAfterOrderCreated() {
        // When
        orderService.createOrder(new CreateOrderCommand("cust-1", List.of(item)));

        // Then (async event)
        await()
            .atMost(Duration.ofSeconds(5))
            .untilAsserted(() ->
                verify(notificationService).sendOrderConfirmation(any(), any())
            );
    }
}
```

### Timeout Assertions
```java
@Test
void shouldCompleteAsyncOperation() {
    CompletableFuture<Result> future = asyncService.process(request);

    await()
        .atMost(Duration.ofSeconds(10))
        .pollInterval(Duration.ofMillis(100))
        .until(future::isDone);

    assertThat(future.get().isSuccess()).isTrue();
}
```

### Waiting for Database State
```java
@Test
void shouldEventuallyUpdateStatus() {
    String orderId = orderService.create(request).id();

    paymentService.processAsync(orderId);

    await()
        .atMost(Duration.ofSeconds(30))
        .pollInterval(Duration.ofSeconds(1))
        .untilAsserted(() -> {
            Order order = orderRepository.findById(orderId).orElseThrow();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
        });
}
```

## Test Configuration

### Test Properties
```yaml
# src/test/resources/application-test.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.springframework.transaction: DEBUG
    org.hibernate.SQL: DEBUG
```

### Test Profile
```java
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceTest {
    // Uses application-test.yml
}
```

### Test Configuration Class
```java
@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public Clock fixedClock() {
        return Clock.fixed(Instant.parse("2024-01-15T10:00:00Z"), ZoneId.of("UTC"));
    }

    @Bean
    @Primary
    public IdGenerator testIdGenerator() {
        return new SequentialIdGenerator();
    }
}

@SpringBootTest
@Import(TestConfig.class)
class OrderServiceTest {
    // Uses fixed clock and sequential ID generator
}
```
