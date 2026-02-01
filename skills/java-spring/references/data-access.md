# Data Access Patterns

## Entity Mapping (No Lombok)

Write JPA entities with explicit getters, protected no-arg constructor, and proper equals/hashCode.

### Entity Example
```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String customerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    private Instant updatedAt;

    @Version
    private Long version;

    // Protected no-arg constructor for JPA
    protected Order() {}

    // Factory method for creation
    public static Order create(String customerId) {
        Order order = new Order();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        order.createdAt = Instant.now();
        return order;
    }

    // Domain behavior
    public void addLine(String productId, int quantity, BigDecimal unitPrice) {
        OrderLine line = new OrderLine(productId, quantity, unitPrice);
        this.lines.add(line);
        this.updatedAt = Instant.now();
    }

    // Explicit getters
    public String getId() { return id; }
    public String getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public List<OrderLine> getLines() { return List.copyOf(lines); }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
    public Long getVersion() { return version; }

    // equals/hashCode on business key or ID (not all fields)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order order)) return false;
        return id != null && id.equals(order.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### Child Entity
```java
@Entity
@Table(name = "order_lines")
public class OrderLine {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String productId;

    @Column(nullable = false)
    private int quantity;

    @Column(nullable = false)
    private BigDecimal unitPrice;

    protected OrderLine() {}

    public OrderLine(String productId, int quantity, BigDecimal unitPrice) {
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }

    public String getId() { return id; }
    public String getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
}
```

## Embedded Value Objects

Use `@Embeddable` for value objects that don't need their own table.

### Embeddable Address
```java
@Embeddable
public class Address {

    @Column(name = "address_street")
    private String street;

    @Column(name = "address_city")
    private String city;

    @Column(name = "address_state")
    private String state;

    @Column(name = "address_postal_code")
    private String postalCode;

    @Column(name = "address_country")
    private String country;

    protected Address() {}

    public Address(String street, String city, String state, String postalCode, String country) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.postalCode = postalCode;
        this.country = country;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getState() { return state; }
    public String getPostalCode() { return postalCode; }
    public String getCountry() { return country; }
}
```

### Using Embedded
```java
@Entity
public class Customer {

    @Id
    private String id;

    private String name;

    @Embedded
    private Address shippingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "billing_street")),
        @AttributeOverride(name = "city", column = @Column(name = "billing_city")),
        @AttributeOverride(name = "state", column = @Column(name = "billing_state")),
        @AttributeOverride(name = "postalCode", column = @Column(name = "billing_postal_code")),
        @AttributeOverride(name = "country", column = @Column(name = "billing_country"))
    })
    private Address billingAddress;
}
```

## Optimistic Locking

Use `@Version` to prevent concurrent modification conflicts.

### Version Field
```java
@Entity
public class Order {

    @Id
    private String id;

    @Version
    private Long version;

    // ... other fields
}
```

### Handling Conflicts
```java
@Service
public class OrderService {

    @Transactional
    public OrderResponse updateOrder(String id, UpdateOrderRequest request) {
        try {
            Order order = orderRepository.findById(id)
                .orElseThrow(() -> new OrderNotFoundException(id));

            order.updateFrom(request);
            // Save triggers version check
            return OrderResponse.from(order);

        } catch (OptimisticLockingFailureException e) {
            throw new ConcurrentModificationException(
                "Order was modified by another user. Please refresh and try again.");
        }
    }
}
```

### REST API Handling with ETag
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
        Order order = orderService.findById(id);
        return ResponseEntity.ok()
            .eTag(String.valueOf(order.getVersion()))
            .body(OrderResponse.from(order));
    }

    @PutMapping("/{id}")
    public ResponseEntity<OrderResponse> updateOrder(
            @PathVariable String id,
            @RequestHeader("If-Match") Long expectedVersion,
            @RequestBody UpdateOrderRequest request) {

        Order order = orderService.findById(id);
        if (!order.getVersion().equals(expectedVersion)) {
            return ResponseEntity.status(HttpStatus.PRECONDITION_FAILED).build();
        }
        return ResponseEntity.ok(orderService.update(id, request));
    }
}
```

## Pessimistic Locking

Use pessimistic locking when you need exclusive access to a row.

### Lock Modes
| Mode | Description | Use Case |
|------|-------------|----------|
| `PESSIMISTIC_READ` | Shared lock, allows concurrent reads | Read-then-update patterns |
| `PESSIMISTIC_WRITE` | Exclusive lock, blocks reads and writes | Critical updates |
| `PESSIMISTIC_FORCE_INCREMENT` | Exclusive lock + version increment | Force version update |

### Repository Methods
```java
public interface AccountRepository extends JpaRepository<Account, String> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") String id);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id IN :ids ORDER BY a.id")
    List<Account> findAllByIdForUpdate(@Param("ids") List<String> ids);
}
```

### Transfer Example
```java
@Service
public class TransferService {

    @Transactional
    public void transfer(String fromId, String toId, BigDecimal amount) {
        // Lock in consistent order to prevent deadlocks
        List<String> orderedIds = Stream.of(fromId, toId).sorted().toList();
        List<Account> accounts = accountRepository.findAllByIdForUpdate(orderedIds);

        Account from = accounts.stream()
            .filter(a -> a.getId().equals(fromId))
            .findFirst()
            .orElseThrow();
        Account to = accounts.stream()
            .filter(a -> a.getId().equals(toId))
            .findFirst()
            .orElseThrow();

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

## N+1 Query Prevention

### The Problem
```java
// BAD: N+1 queries
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order order : orders) {
    order.getLines().size(); // N queries (one per order)
}
```

### Solution 1: JOIN FETCH
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.customerId = :customerId")
    List<Order> findByCustomerIdWithLines(@Param("customerId") String customerId);

    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.lines JOIN FETCH o.customer")
    List<Order> findAllWithLinesAndCustomer();
}
```

### Solution 2: @EntityGraph
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @EntityGraph(attributePaths = {"lines", "customer"})
    List<Order> findByStatus(OrderStatus status);

    @EntityGraph(attributePaths = {"lines"})
    Optional<Order> findById(String id);
}
```

### Solution 3: DTO Projections (Best for Read-Only)

#### Interface Projection
```java
public interface OrderSummary {
    String getId();
    String getCustomerId();
    String getStatus();
    BigDecimal getTotal();
}

public interface OrderRepository extends JpaRepository<Order, String> {
    List<OrderSummary> findSummaryByCustomerId(String customerId);
}
```

#### Record Projection
```java
public record OrderSummaryDto(String id, String customerId, String status, BigDecimal total) {}

public interface OrderRepository extends JpaRepository<Order, String> {

    @Query("""
        SELECT new com.acme.order.dto.OrderSummaryDto(o.id, o.customerId, o.status, o.total)
        FROM Order o
        WHERE o.customerId = :customerId
        """)
    List<OrderSummaryDto> findSummaryByCustomerId(@Param("customerId") String customerId);
}
```

### Detecting N+1 in Tests
```yaml
# application-test.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
```

## Repository Patterns

### Query Methods
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    // Derived queries
    List<Order> findByCustomerId(String customerId);
    List<Order> findByStatusIn(List<OrderStatus> statuses);
    List<Order> findByCreatedAtBetween(Instant start, Instant end);
    Optional<Order> findFirstByCustomerIdOrderByCreatedAtDesc(String customerId);
    boolean existsByCustomerIdAndStatus(String customerId, OrderStatus status);
    long countByStatus(OrderStatus status);

    // Paging
    Page<Order> findByCustomerId(String customerId, Pageable pageable);
    Slice<Order> findByStatus(OrderStatus status, Pageable pageable);
}
```

### JPQL Queries
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @Query("""
        SELECT o FROM Order o
        WHERE o.status = :status
        AND o.createdAt >= :since
        ORDER BY o.createdAt DESC
        """)
    List<Order> findRecentByStatus(
        @Param("status") OrderStatus status,
        @Param("since") Instant since);

    @Query("""
        SELECT o FROM Order o
        WHERE o.customerId = :customerId
        AND (:status IS NULL OR o.status = :status)
        """)
    List<Order> findByCustomerIdAndOptionalStatus(
        @Param("customerId") String customerId,
        @Param("status") OrderStatus status);
}
```

### Native Queries
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @Query(value = """
        SELECT o.* FROM orders o
        WHERE o.status = :status
        AND o.created_at >= NOW() - INTERVAL '30 days'
        """, nativeQuery = true)
    List<Order> findRecentByStatusNative(@Param("status") String status);
}
```

### Modifying Queries
```java
public interface OrderRepository extends JpaRepository<Order, String> {

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    int updateStatus(@Param("id") String id, @Param("status") OrderStatus status);

    @Modifying
    @Query("DELETE FROM Order o WHERE o.status = :status AND o.createdAt < :before")
    int deleteOldByStatus(@Param("status") OrderStatus status, @Param("before") Instant before);
}
```

## Auditing

### Enable Auditing
```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

### Auditable Entity
```java
@Entity
public class Order {

    @Id
    private String id;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    // ... other fields and methods
}
```

### Auditable Base Class
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
    public String getCreatedBy() { return createdBy; }
    public String getUpdatedBy() { return updatedBy; }
}

@Entity
public class Order extends AuditableEntity {
    // Order-specific fields
}
```

## Schema Migration with Flyway

### Project Structure
```
src/main/resources/
└── db/migration/
    ├── V1__create_customers_table.sql
    ├── V2__create_orders_table.sql
    ├── V3__add_order_status_index.sql
    └── V4__add_shipping_address.sql
```

### Naming Convention
- `V{version}__{description}.sql` - Versioned migrations (run once)
- `R__{description}.sql` - Repeatable migrations (run on change)
- Double underscore (`__`) between version and description

### Example Migrations
```sql
-- V1__create_customers_table.sql
CREATE TABLE customers (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);

-- V2__create_orders_table.sql
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    customer_id VARCHAR(36) NOT NULL REFERENCES customers(id),
    status VARCHAR(50) NOT NULL,
    total DECIMAL(19,2),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    version BIGINT DEFAULT 0
);

CREATE TABLE order_lines (
    id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id VARCHAR(36) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(19,2) NOT NULL
);

-- V3__add_order_status_index.sql
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer_created ON orders(customer_id, created_at DESC);
```

### Configuration
```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    validate-on-migrate: true
```

### Best Practices
1. **Never modify** existing migrations once deployed
2. **Make migrations idempotent** when possible
3. **Test migrations** before production deployment
4. **Keep migrations small** and focused
5. **Use transactions** (default for Flyway)
