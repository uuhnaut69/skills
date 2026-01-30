---
name: spring-boot
description: Spring Boot development patterns, best practices, and Spring Boot 4 features. Covers REST APIs, dependency injection, testing strategies, error handling with ProblemDetail, and modern Java patterns like records for DTOs.
---

# Spring Boot Development Guide

## Quick Reference

### Dependency Injection
Always use **constructor injection**, never field injection:
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository orderRepository, PaymentClient paymentClient) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
    }
}
```

### DTOs with Records
Use Java records for immutable DTOs:
```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<OrderItem> items
) {}

public record OrderResponse(
    String id,
    String status,
    List<OrderItem> items,
    Instant createdAt
) {}
```

### REST Controllers
Use `@RestController` with explicit mappings:
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        return orderService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }
}
```

### Error Handling with ProblemDetail
Use `@RestControllerAdvice` with RFC 7807 ProblemDetail:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Order Not Found");
        problem.setProperty("orderId", ex.getOrderId());
        return problem;
    }
}
```

### Test Slices
Choose the right test scope:
- `@WebMvcTest` - Controller layer only (fast, no server)
- `@DataJpaTest` - Repository layer with H2
- `@SpringBootTest` - Full integration tests (use sparingly)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockitoBean OrderService orderService;

    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.findById("123")).thenReturn(testOrder);

        mockMvc.perform(get("/api/orders/123"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("123"));
    }
}
```

## Reference Documents

### When to load additional references:

**Load `references/spring-boot-4.md` when:**
- Working with Spring Boot 4.x projects
- Using Jackson 3 / JsonMapper
- Implementing null safety with JSpecify
- Using RestTestClient for testing
- Implementing resilience patterns (@Retryable, @ConcurrencyLimit)

**Load `references/spring-boot-best-practices.md` when:**
- Designing new Spring Boot applications
- Refactoring existing code for better patterns
- Setting up HTTP Interface Clients or RestClient
- Configuring observability with Micrometer
- Organizing package structure
