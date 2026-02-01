# Spring Boot Best Practices

## Interface Overuse

**Don't create interfaces for single implementations.** Only create interfaces when you have multiple implementations or explicit architectural reasons, not by default.

### Anti-Pattern (Unnecessary Abstraction)
```java
// UserService.java
public interface UserService {
    User findById(String id);
    User save(User user);
}

// UserServiceImpl.java
@Service
public class UserServiceImpl implements UserService {
    // Only implementation - interface adds no value
}
```

### Correct Pattern
```java
@Service
public class UserService {
    public User findById(String id) { ... }
    public User save(User user) { ... }
}
```

### When Interfaces Make Sense
- Multiple implementations (e.g., `PaymentProcessor` with `StripePaymentProcessor`, `PayPalPaymentProcessor`)
- Explicit contract for external consumers
- Testing requirements that genuinely benefit from abstraction

## Constructor Injection

**Always use constructor injection.** Never use field injection (`@Autowired` on fields).

### Why Constructor Injection
- Immutable dependencies (final fields)
- Required dependencies are explicit
- Easier to test (no reflection needed)
- Fails fast if dependencies are missing

### Correct Pattern
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;
    private final EventPublisher eventPublisher;

    // Single constructor - @Autowired is optional
    public OrderService(
            OrderRepository orderRepository,
            PaymentClient paymentClient,
            EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
        this.eventPublisher = eventPublisher;
    }
}
```

### Anti-Pattern (Never Do This)
```java
@Service
public class OrderService {
    @Autowired  // BAD: Field injection
    private OrderRepository orderRepository;

    @Autowired  // BAD: Field injection
    private PaymentClient paymentClient;
}
```

## REST API Design

### Controller Structure
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping
    public List<OrderResponse> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return orderService.findAll(PageRequest.of(page, size));
    }

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        return orderService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @PutMapping("/{id}")
    public OrderResponse updateOrder(
            @PathVariable String id,
            @Valid @RequestBody UpdateOrderRequest request) {
        return orderService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteOrder(@PathVariable String id) {
        orderService.delete(id);
    }
}
```

### Response Status Codes
- `200 OK` - Successful GET, PUT
- `201 Created` - Successful POST creating resource
- `204 No Content` - Successful DELETE
- `400 Bad Request` - Validation errors
- `404 Not Found` - Resource not found
- `409 Conflict` - Business rule violation

## Records for DTOs

Use Java records for request/response DTOs. They're immutable, concise, and work well with validation.

### Request DTOs
```java
public record CreateOrderRequest(
    @NotBlank(message = "Customer ID is required")
    String customerId,

    @NotEmpty(message = "Order must have at least one item")
    List<@Valid OrderItemRequest> items,

    @Email(message = "Invalid email format")
    String notificationEmail
) {}

public record OrderItemRequest(
    @NotBlank String productId,
    @Positive int quantity
) {}
```

### Response DTOs
```java
public record OrderResponse(
    String id,
    String customerId,
    OrderStatus status,
    List<OrderItemResponse> items,
    BigDecimal total,
    Instant createdAt,
    Instant updatedAt
) {
    // Factory method for mapping from entity
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId(),
            order.getCustomerId(),
            order.getStatus(),
            order.getItems().stream().map(OrderItemResponse::from).toList(),
            order.calculateTotal(),
            order.getCreatedAt(),
            order.getUpdatedAt()
        );
    }
}
```

## Error Handling with ProblemDetail

Use RFC 7807 Problem Details for consistent error responses.

### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setProperty("resourceType", ex.getResourceType());
        problem.setProperty("resourceId", ex.getResourceId());
        return problem;
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ProblemDetail handleBusinessRule(BusinessRuleException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, ex.getMessage());
        problem.setTitle("Business Rule Violation");
        problem.setProperty("ruleCode", ex.getRuleCode());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation Error");

        var errors = ex.getBindingResult().getFieldErrors().stream()
            .map(error -> Map.of(
                "field", error.getField(),
                "message", error.getDefaultMessage(),
                "rejected", String.valueOf(error.getRejectedValue())
            ))
            .toList();
        problem.setProperty("errors", errors);
        return problem;
    }
}
```

### Custom Exceptions
```java
public class ResourceNotFoundException extends RuntimeException {
    private final String resourceType;
    private final String resourceId;

    public ResourceNotFoundException(String resourceType, String resourceId) {
        super("%s with id '%s' not found".formatted(resourceType, resourceId));
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }

    public String getResourceType() { return resourceType; }
    public String getResourceId() { return resourceId; }
}
```

## @ConfigurationProperties

Use type-safe configuration over `@Value`.

### Define Properties Class
```java
@ConfigurationProperties(prefix = "app.orders")
public record OrderProperties(
    int maxItemsPerOrder,
    Duration processingTimeout,
    RetryConfig retry
) {
    public record RetryConfig(
        int maxAttempts,
        Duration initialDelay
    ) {}
}
```

### Enable and Use
```java
@SpringBootApplication
@EnableConfigurationProperties(OrderProperties.class)
public class Application {}

@Service
public class OrderService {
    private final OrderProperties properties;

    public OrderService(OrderProperties properties) {
        this.properties = properties;
    }

    public void process(Order order) {
        if (order.getItems().size() > properties.maxItemsPerOrder()) {
            throw new TooManyItemsException();
        }
        // use properties.processingTimeout(), properties.retry().maxAttempts(), etc.
    }
}
```

### application.yml
```yaml
app:
  orders:
    max-items-per-order: 50
    processing-timeout: 30s
    retry:
      max-attempts: 3
      initial-delay: 1s
```

## RestClient (Imperative HTTP)

For more control or one-off requests.

```java
@Service
public class NotificationService {
    private final RestClient restClient;

    public NotificationService(RestClient.Builder builder, NotificationProperties props) {
        this.restClient = builder
            .baseUrl(props.baseUrl())
            .build();
    }

    public void sendNotification(Notification notification) {
        restClient.post()
            .uri("/notifications")
            .contentType(MediaType.APPLICATION_JSON)
            .body(notification)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                throw new NotificationFailedException("Client error: " + response.getStatusCode());
            })
            .toBodilessEntity();
    }

    public Optional<NotificationStatus> getStatus(String notificationId) {
        return restClient.get()
            .uri("/notifications/{id}/status", notificationId)
            .retrieve()
            .onStatus(status -> status.value() == 404, (req, res) -> {})
            .body(new ParameterizedTypeReference<Optional<NotificationStatus>>() {});
    }
}
```

## Observability with Micrometer

### Custom Metrics
```java
@Service
public class OrderService {
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(registry);

        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Time to process orders")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public OrderResponse create(CreateOrderRequest request) {
        return orderProcessingTime.record(() -> {
            var order = processOrder(request);
            ordersCreated.increment();
            return OrderResponse.from(order);
        });
    }
}
```

### @Observed for Tracing
```java
@Service
public class OrderService {

    @Observed(name = "order.create",
              contextualName = "creating-order",
              lowCardinalityKeyValues = {"service", "order-service"})
    public OrderResponse create(CreateOrderRequest request) {
        // Automatically creates span and metrics
        return processOrder(request);
    }
}
```

### Configuration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles-histogram:
        http.server.requests: true
  tracing:
    sampling:
      probability: 1.0
```

## Structured Logging (Spring Boot 3.4+)

Enable JSON-formatted logs for better log aggregation and analysis.

### Enable Structured Logging
```yaml
logging:
  structured:
    format:
      console: ecs  # or logstash, gelf
```

### Available Formats
- `ecs` - Elastic Common Schema
- `logstash` - Logstash JSON format
- `gelf` - Graylog Extended Log Format

### Custom Structured Fields
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void processOrder(String orderId, String customerId) {
        try (var ignored = MDC.putCloseable("orderId", orderId);
             var ignored2 = MDC.putCloseable("customerId", customerId)) {
            log.info("Processing order");
            // MDC fields automatically included in structured output
        }
    }
}
```

### Configuration for Production
```yaml
logging:
  structured:
    format:
      console: ecs
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    correlation: "[${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```
