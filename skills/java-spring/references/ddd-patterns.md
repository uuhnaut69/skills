# DDD Patterns for Spring Applications

## Package Structure (Business-First Modules)

### Anti-Pattern: Layer-Based Packages
```
com.acme.app/
├── controllers/     # All controllers mixed together
│   ├── OrderController.java
│   ├── CustomerController.java
│   └── PaymentController.java
├── services/        # All services mixed together
├── repositories/    # All repositories mixed together
└── entities/        # All entities mixed together
```

**Problems:**
- Related code scattered across packages
- Hard to understand feature boundaries
- Difficult to extract microservices
- High coupling between features

### Correct: Business-First Modules
```
com.acme.app/
├── order/
│   ├── domain/
│   │   ├── Order.java           # Aggregate root
│   │   ├── OrderLine.java       # Entity within aggregate
│   │   ├── OrderStatus.java     # Enum
│   │   ├── OrderId.java         # Value object
│   │   └── OrderRepository.java # Port (interface)
│   ├── application/
│   │   ├── OrderService.java    # Use cases
│   │   ├── OrderMapper.java     # DTO mapping
│   │   └── dto/
│   │       ├── CreateOrderCommand.java
│   │       └── OrderResponse.java
│   └── infrastructure/
│       ├── JpaOrderRepository.java  # Adapter
│       └── OrderController.java
├── customer/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
├── payment/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
└── shared/
    ├── domain/
    │   └── Money.java           # Shared value object
    └── infrastructure/
        └── GlobalExceptionHandler.java
```

**Benefits:**
- Related code is together
- Clear feature boundaries
- Easy to extract to microservices
- Low coupling between modules

## Aggregates

An aggregate is a cluster of domain objects treated as a single unit. The aggregate root is the only entry point.

### Aggregate Root Example
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private String id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    @Embedded
    private Money total;

    @Version
    private Long version;

    protected Order() {} // JPA

    // Factory method with validation
    public static Order create(String customerId, List<OrderLine> lines) {
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one line");
        }
        Order order = new Order();
        order.id = UUID.randomUUID().toString();
        order.status = OrderStatus.PENDING;
        lines.forEach(order::addLine);
        order.recalculateTotal();
        return order;
    }

    // Domain behavior, not just setters
    public void addLine(OrderLine line) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        lines.add(line);
        recalculateTotal();
    }

    public void removeLine(String productId) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        lines.removeIf(line -> line.getProductId().equals(productId));
        recalculateTotal();
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Order already confirmed or cancelled");
        }
        if (lines.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("Cannot cancel shipped order");
        }
        this.status = OrderStatus.CANCELLED;
    }

    private void recalculateTotal() {
        this.total = lines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // Getters only, no setters for domain integrity
    public String getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public List<OrderLine> getLines() { return List.copyOf(lines); } // Defensive copy
    public Money getTotal() { return total; }
}
```

### Key Principles
1. **Factory methods** for creation with validation
2. **Domain behavior** in methods, not setters
3. **Encapsulated collections** with defensive copies
4. **Invariants enforced** within the aggregate
5. **No Lombok**—explicit getters, no setters for protected fields

## Value Objects

Value objects are immutable, identified by their attributes, not by an ID. Use Java records.

### Money Value Object
```java
public record Money(BigDecimal amount, Currency currency) {

    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("USD"));

    public Money {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (currency == null) {
            throw new IllegalArgumentException("Currency cannot be null");
        }
        if (amount.scale() > 2) {
            amount = amount.setScale(2, RoundingMode.HALF_UP);
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
}
```

### OrderId Value Object
```java
public record OrderId(String value) {

    public OrderId {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Order ID cannot be blank");
        }
    }

    public static OrderId generate() {
        return new OrderId(UUID.randomUUID().toString());
    }

    @Override
    public String toString() {
        return value;
    }
}
```

### Address Value Object
```java
public record Address(
    String street,
    String city,
    String state,
    String postalCode,
    String country
) {
    public Address {
        if (street == null || street.isBlank()) {
            throw new IllegalArgumentException("Street is required");
        }
        if (city == null || city.isBlank()) {
            throw new IllegalArgumentException("City is required");
        }
        if (country == null || country.isBlank()) {
            throw new IllegalArgumentException("Country is required");
        }
    }
}
```

## Domain Events

Use domain events for side effects that should happen after a transaction commits.

### Event Definition
```java
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    Money total,
    Instant occurredAt
) {
    public OrderPlacedEvent(Order order) {
        this(order.getId(), order.getCustomerId(), order.getTotal(), Instant.now());
    }
}
```

### Publishing Events
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository, ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public OrderResponse placeOrder(CreateOrderCommand command) {
        Order order = Order.create(command.customerId(), command.lines());
        order.confirm();
        orderRepository.save(order);

        // Publish event after save (still within transaction)
        eventPublisher.publishEvent(new OrderPlacedEvent(order));

        return OrderResponse.from(order);
    }
}
```

### Listening to Events
```java
@Component
public class OrderEventHandlers {
    private final BillingService billingService;
    private final NotificationService notificationService;

    public OrderEventHandlers(BillingService billingService, NotificationService notificationService) {
        this.billingService = billingService;
        this.notificationService = notificationService;
    }

    // Runs AFTER the transaction commits successfully
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlaced(OrderPlacedEvent event) {
        billingService.createInvoice(event.orderId(), event.total());
    }

    // Async processing for non-critical side effects
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendOrderConfirmation(OrderPlacedEvent event) {
        notificationService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}
```

### Event Listener Phases
| Phase | Description | Use Case |
|-------|-------------|----------|
| `BEFORE_COMMIT` | Runs before transaction commits | Validation, enrichment |
| `AFTER_COMMIT` | Runs after successful commit | External calls, notifications |
| `AFTER_ROLLBACK` | Runs after rollback | Compensating actions |
| `AFTER_COMPLETION` | Runs after commit or rollback | Cleanup |

## Repository Pattern

The repository is a port (interface) in the domain layer, with an adapter (implementation) in infrastructure.

### Port (Domain Layer)
```java
// In domain package - no Spring dependencies
public interface OrderRepository {

    Order save(Order order);

    Optional<Order> findById(String id);

    List<Order> findByCustomerId(String customerId);

    List<Order> findByStatus(OrderStatus status);

    void delete(Order order);
}
```

### Adapter (Infrastructure Layer)
```java
// In infrastructure package - Spring Data JPA implementation
public interface JpaOrderRepository extends JpaRepository<Order, String>, OrderRepository {

    // Spring Data derives query from method name
    List<Order> findByCustomerId(String customerId);

    List<Order> findByStatus(OrderStatus status);

    // Custom query for complex cases
    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") String id);
}
```

### Why This Pattern?
- Domain layer stays pure (no framework dependencies)
- Easy to test with in-memory implementations
- Infrastructure concerns isolated
- Can swap implementations (JPA → MongoDB) without domain changes

## Application Services

Application services orchestrate use cases. They coordinate domain objects and infrastructure.

### Service Structure
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final OrderMapper orderMapper;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(
            OrderRepository orderRepository,
            CustomerRepository customerRepository,
            OrderMapper orderMapper,
            ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
        this.orderMapper = orderMapper;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public OrderResponse createOrder(CreateOrderCommand command) {
        // Validate customer exists
        Customer customer = customerRepository.findById(command.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));

        // Create aggregate through factory method
        List<OrderLine> lines = orderMapper.toOrderLines(command.items());
        Order order = Order.create(customer.getId(), lines);

        // Persist
        Order saved = orderRepository.save(order);

        // Publish domain event
        eventPublisher.publishEvent(new OrderCreatedEvent(saved));

        return OrderResponse.from(saved);
    }

    @Transactional
    public OrderResponse confirmOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        // Domain logic in aggregate
        order.confirm();

        // No explicit save needed - dirty checking
        eventPublisher.publishEvent(new OrderConfirmedEvent(order));

        return OrderResponse.from(order);
    }

    @Transactional(readOnly = true)
    public OrderResponse findById(String orderId) {
        return orderRepository.findById(orderId)
            .map(OrderResponse::from)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

### Key Principles
1. **Transaction boundary** on public methods
2. **Orchestration only**—domain logic stays in aggregates
3. **DTOs at boundaries**—never expose entities
4. **Event publishing** for side effects
5. **readOnly = true** for queries (performance optimization)

## API Layer

Controllers are thin—just routing, validation triggering, and response mapping.

### Controller Structure
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        return orderService.findById(id);
    }

    @GetMapping
    public List<OrderResponse> listOrders(
            @RequestParam(required = false) String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        if (customerId != null) {
            return orderService.findByCustomerId(customerId);
        }
        return orderService.findAll(PageRequest.of(page, size));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.createOrder(CreateOrderCommand.from(request));
    }

    @PostMapping("/{id}/confirm")
    public OrderResponse confirmOrder(@PathVariable String id) {
        return orderService.confirmOrder(id);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void cancelOrder(@PathVariable String id) {
        orderService.cancelOrder(id);
    }
}
```

### Request/Response Records
```java
// Request DTO with validation
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemRequest> items
) {
    public record OrderItemRequest(
        @NotBlank String productId,
        @Positive int quantity
    ) {}
}

// Command for service layer (may add defaults, transformations)
public record CreateOrderCommand(
    String customerId,
    List<OrderLineData> items
) {
    public static CreateOrderCommand from(CreateOrderRequest request) {
        return new CreateOrderCommand(
            request.customerId(),
            request.items().stream()
                .map(i -> new OrderLineData(i.productId(), i.quantity()))
                .toList()
        );
    }
}

// Response DTO
public record OrderResponse(
    String id,
    String customerId,
    String status,
    List<OrderLineResponse> items,
    BigDecimal total,
    Instant createdAt
) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId(),
            order.getCustomerId(),
            order.getStatus().name(),
            order.getLines().stream().map(OrderLineResponse::from).toList(),
            order.getTotal().amount(),
            order.getCreatedAt()
        );
    }
}
```

### Controller Responsibilities
| Do | Don't |
|----|-------|
| Route HTTP requests | Contain business logic |
| Trigger validation (`@Valid`) | Access repositories directly |
| Map to/from DTOs | Know about entities |
| Set response status codes | Manage transactions |
| Handle request parameters | Call other controllers |
