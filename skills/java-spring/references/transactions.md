# Transaction Management

## Transaction Boundaries

Place `@Transactional` on service layer public methods, never on repository methods.

### Correct Placement
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    @Transactional
    public OrderResponse createOrder(CreateOrderCommand command) {
        Order order = Order.create(command.customerId());
        command.items().forEach(item ->
            order.addLine(item.productId(), item.quantity(), item.unitPrice()));
        orderRepository.save(order);
        return OrderResponse.from(order);
    }

    @Transactional(readOnly = true)
    public OrderResponse findById(String id) {
        return orderRepository.findById(id)
            .map(OrderResponse::from)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

### Repository Layer (No @Transactional)
```java
// Spring Data JPA handles transaction for each method automatically
public interface OrderRepository extends JpaRepository<Order, String> {
    // Don't add @Transactional here
    List<Order> findByCustomerId(String customerId);
}
```

## Propagation Modes

| Propagation | Behavior | Use Case |
|-------------|----------|----------|
| `REQUIRED` (default) | Join existing or create new | Most service methods |
| `REQUIRES_NEW` | Always create new (suspend existing) | Audit logs, independent operations |
| `NESTED` | Create savepoint within existing | Partial rollback support |
| `MANDATORY` | Must have existing, error if none | Methods that should never start tx |
| `SUPPORTS` | Join if exists, non-tx otherwise | Read operations that can work either way |
| `NOT_SUPPORTED` | Execute non-tx (suspend existing) | External calls during transaction |
| `NEVER` | Error if transaction exists | Validation that tx shouldn't exist |

### REQUIRES_NEW Example
```java
@Service
public class AuditService {
    private final AuditLogRepository auditLogRepository;

    public AuditService(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    // Always commits, even if calling transaction rolls back
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, String entityId, String userId) {
        AuditLog log = new AuditLog(action, entityId, userId, Instant.now());
        auditLogRepository.save(log);
    }
}

@Service
public class OrderService {
    private final AuditService auditService;

    @Transactional
    public void processOrder(String orderId) {
        auditService.logAction("PROCESS_START", orderId, getCurrentUser());

        try {
            // Business logic that might fail
            doProcessing(orderId);
            auditService.logAction("PROCESS_SUCCESS", orderId, getCurrentUser());
        } catch (Exception e) {
            auditService.logAction("PROCESS_FAILED", orderId, getCurrentUser());
            throw e; // Main tx rolls back, but audit logs are committed
        }
    }
}
```

### NESTED Example
```java
@Service
public class BatchService {

    @Transactional
    public BatchResult processBatch(List<Item> items) {
        List<String> succeeded = new ArrayList<>();
        List<String> failed = new ArrayList<>();

        for (Item item : items) {
            try {
                processItemWithSavepoint(item);
                succeeded.add(item.getId());
            } catch (Exception e) {
                // Savepoint rolled back, but outer tx continues
                failed.add(item.getId());
            }
        }

        return new BatchResult(succeeded, failed);
    }

    @Transactional(propagation = Propagation.NESTED)
    public void processItemWithSavepoint(Item item) {
        // If this fails, only this item's changes roll back
        itemRepository.save(item);
        externalService.notify(item); // Might throw
    }
}
```

## Rollback Rules

### Default Behavior
- **Unchecked exceptions** (RuntimeException): Rollback
- **Checked exceptions**: Commit (no rollback)
- **Errors**: Rollback

### Configure Rollback
```java
@Service
public class PaymentService {

    // Rollback on all exceptions
    @Transactional(rollbackFor = Exception.class)
    public void processPayment(PaymentRequest request) throws PaymentException {
        // Even checked PaymentException will cause rollback
    }

    // Don't rollback on specific exception
    @Transactional(noRollbackFor = InsufficientFundsException.class)
    public void transferFunds(TransferRequest request) {
        // InsufficientFundsException won't cause rollback
    }

    // Combine both
    @Transactional(
        rollbackFor = Exception.class,
        noRollbackFor = {ValidationException.class, WarningException.class}
    )
    public void complexOperation(Request request) {
        // Rollback all except ValidationException and WarningException
    }
}
```

### Programmatic Rollback
```java
@Service
public class OrderService {

    @Transactional
    public OrderResult processOrder(OrderRequest request) {
        Order order = createOrder(request);

        ValidationResult validation = validateOrder(order);
        if (!validation.isValid()) {
            // Mark for rollback without throwing exception
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            return OrderResult.failed(validation.getErrors());
        }

        return OrderResult.success(order.getId());
    }
}
```

## Anti-Patterns

### Self-Invocation Bypasses Proxy
```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(String orderId) {
        // Business logic
    }

    public void processMultipleOrders(List<String> orderIds) {
        for (String orderId : orderIds) {
            // BAD: Direct call bypasses @Transactional proxy!
            processOrder(orderId);
        }
    }
}
```

#### Fix 1: Inject Self
```java
@Service
public class OrderService {
    private final OrderService self;

    public OrderService(@Lazy OrderService self) {
        this.self = self;
    }

    @Transactional
    public void processOrder(String orderId) {
        // Business logic
    }

    public void processMultipleOrders(List<String> orderIds) {
        for (String orderId : orderIds) {
            self.processOrder(orderId); // Goes through proxy
        }
    }
}
```

#### Fix 2: Extract to Separate Service
```java
@Service
public class OrderProcessor {

    @Transactional
    public void processOrder(String orderId) {
        // Business logic
    }
}

@Service
public class BatchOrderService {
    private final OrderProcessor orderProcessor;

    public BatchOrderService(OrderProcessor orderProcessor) {
        this.orderProcessor = orderProcessor;
    }

    public void processMultipleOrders(List<String> orderIds) {
        for (String orderId : orderIds) {
            orderProcessor.processOrder(orderId); // Goes through proxy
        }
    }
}
```

### Swallowing Exceptions
```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(String orderId) {
        try {
            riskyOperation();
        } catch (Exception e) {
            // BAD: Exception swallowed, transaction commits despite failure
            log.error("Failed", e);
        }
    }
}

// Fix: Re-throw or mark for rollback
@Transactional
public void processOrder(String orderId) {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);
        throw e; // Transaction will rollback
    }
}
```

## Persistence Concepts

### Dirty Checking
JPA automatically detects changes to managed entities—no explicit save needed.

```java
@Service
public class OrderService {

    @Transactional
    public void updateOrderStatus(String orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.setStatus(newStatus); // Entity is "dirty"

        // No save() needed! Hibernate detects the change
        // and generates UPDATE on transaction commit
    }
}
```

### Flush vs Commit
| Operation | Description | When It Happens |
|-----------|-------------|-----------------|
| **Flush** | Sync pending changes to database (SQL executed) | Before queries, explicit call, tx commit |
| **Commit** | Make changes permanent, release locks | End of transaction |

```java
@Service
public class OrderService {

    @Transactional
    public void createOrderWithId(CreateOrderCommand command) {
        Order order = Order.create(command.customerId());
        orderRepository.save(order);

        // ID might be null if using @GeneratedValue with IDENTITY
        // Force flush to get the ID immediately
        orderRepository.flush();

        // Now order.getId() is guaranteed to have the generated value
        auditService.log("Created order: " + order.getId());
    }
}
```

### When to Use saveAndFlush
```java
// Use saveAndFlush when you need the ID immediately
@Transactional
public String createAndReturnId(CreateRequest request) {
    Entity entity = new Entity(request);
    Entity saved = repository.saveAndFlush(entity);
    return saved.getId(); // Guaranteed to have ID
}

// Regular save is sufficient when ID isn't needed until commit
@Transactional
public void createWithoutImmediateId(CreateRequest request) {
    Entity entity = new Entity(request);
    repository.save(entity);
    // ID populated at commit time
}
```

## Performance Optimizations

### readOnly = true
```java
@Service
public class OrderQueryService {

    // Hibernate skips dirty checking, may use read replica
    @Transactional(readOnly = true)
    public List<OrderSummary> findByCustomer(String customerId) {
        return orderRepository.findSummaryByCustomerId(customerId);
    }

    @Transactional(readOnly = true)
    public Page<OrderResponse> findAll(Pageable pageable) {
        return orderRepository.findAll(pageable)
            .map(OrderResponse::from);
    }
}
```

### Keep Transactions Short
```java
// BAD: Long transaction with external call
@Transactional
public void processOrder(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(PROCESSING);

    // BAD: HTTP call inside transaction holds DB connection
    PaymentResult result = paymentClient.charge(order.getTotal());

    order.setPaymentId(result.getId());
}

// GOOD: External calls outside transaction
public void processOrder(String orderId) {
    Order order = prepareForProcessing(orderId);

    // HTTP call outside transaction
    PaymentResult result = paymentClient.charge(order.getTotal());

    completeProcessing(orderId, result);
}

@Transactional
Order prepareForProcessing(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(PROCESSING);
    return order;
}

@Transactional
void completeProcessing(String orderId, PaymentResult result) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setPaymentId(result.getId());
    order.setStatus(PAID);
}
```

### Timeout
```java
@Service
public class ReportService {

    @Transactional(timeout = 30) // 30 seconds timeout
    public Report generateReport(ReportRequest request) {
        // Long-running query
        return reportRepository.generateComplexReport(request);
    }
}
```

## LazyInitializationException

### The Problem
```java
@Service
public class OrderService {

    @Transactional
    public Order findById(String id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

@RestController
public class OrderController {
    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        // LazyInitializationException! Transaction already closed
        return OrderResponse.from(order.getLines()); // lines is LAZY
    }
}
```

### Solution 1: DTO Projection (Recommended)
```java
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public OrderResponse findById(String id) {
        Order order = orderRepository.findById(id).orElseThrow();
        return OrderResponse.from(order); // Map inside transaction
    }
}
```

### Solution 2: Fetch in Query
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") String id);
}
```

### Solution 3: @EntityGraph
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @EntityGraph(attributePaths = {"lines"})
    Optional<Order> findById(String id);
}
```

### Anti-Pattern: Open Session in View
```yaml
# DON'T do this - it keeps DB connections open too long
spring:
  jpa:
    open-in-view: false  # Keep this disabled (default in SB 2.0+)
```

## Multiple DataSources

### Configuration
```java
@Configuration
public class DataSourceConfig {

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    public PlatformTransactionManager secondaryTransactionManager(
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

### Explicit Transaction Manager
```java
@Service
public class ReportService {

    @Transactional(transactionManager = "secondaryTransactionManager", readOnly = true)
    public Report generateFromReplica() {
        // Uses secondary (read replica) datasource
        return reportRepository.generate();
    }
}
```

### Multiple Databases in One Transaction
For distributed transactions across multiple databases, consider:
1. **Saga pattern** - Choreographed compensation
2. **Transactional outbox** - Reliable event publishing
3. **Two-phase commit** - JTA (last resort, adds complexity)

```java
// Saga pattern example
@Service
public class OrderSaga {

    @Transactional // Primary DB
    public void createOrder(CreateOrderCommand command) {
        Order order = orderRepository.save(Order.create(command));

        try {
            inventoryClient.reserve(order.getItems()); // External call
        } catch (Exception e) {
            // Compensate: Delete order
            orderRepository.delete(order);
            throw new OrderCreationFailedException(e);
        }
    }
}
```
