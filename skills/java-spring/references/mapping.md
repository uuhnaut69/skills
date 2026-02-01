# DTO Mapping Patterns

## Layer Responsibilities

| Layer | Knows DTOs | Knows Entities | Calls Repository | Handles Mapping |
|-------|-----------|----------------|------------------|-----------------|
| Controller | Yes | No | No | Request → Command |
| Service | Yes | Yes | Yes | Command → Entity, Entity → Response |
| Mapper | Yes | Yes | No | Stateless conversions |
| Domain | No | N/A | No | None |

### Data Flow
```
Request DTO → Controller → Command → Service → Entity
                                        ↓
Response DTO ← Controller ← Response ← Service ← Entity
```

## Mapping Approaches

### When to Use MapStruct
- Multiple fields to map
- Nested object mappings
- Frequent entity-DTO conversions
- Type conversions needed

### When to Use Record Factory Methods
- Simple mappings (< 5 fields)
- One-off mappings
- Response DTOs with straightforward field access
- Command objects from requests

## MapStruct Setup

### Maven Dependency
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

### Basic Mapper
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface OrderMapper {

    // Fields with same name map automatically
    OrderResponse toResponse(Order order);

    // Explicit mapping for different names
    @Mapping(source = "productId", target = "product")
    @Mapping(source = "unitPrice", target = "price")
    OrderLineResponse toLineResponse(OrderLine line);

    // Collection mapping (uses element mapper)
    List<OrderLineResponse> toLineResponses(List<OrderLine> lines);
}
```

### Request to Command Mapping
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface OrderCommandMapper {

    CreateOrderCommand toCommand(CreateOrderRequest request);

    OrderLineData toLineData(OrderItemRequest item);

    List<OrderLineData> toLineDataList(List<OrderItemRequest> items);
}
```

### Entity to Response with Nested Mapping
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface OrderMapper {

    @Mapping(source = "status", target = "status", qualifiedByName = "statusToString")
    @Mapping(source = "total.amount", target = "totalAmount")
    @Mapping(source = "total.currency", target = "currency")
    OrderResponse toResponse(Order order);

    @Named("statusToString")
    default String statusToString(OrderStatus status) {
        return status.name().toLowerCase();
    }
}
```

### Update Existing Entity
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING,
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface OrderMapper {

    // Updates existing entity, ignoring null values in DTO
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    void updateFromRequest(UpdateOrderRequest request, @MappingTarget Order order);
}
```

### Ignoring Server-Owned Fields
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface OrderMapper {

    // Never let clients set these fields
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "version", ignore = true)
    Order toEntity(CreateOrderRequest request);
}
```

### Reference Resolution
```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface OrderMapper {

    @Mapping(target = "customer", source = "customerId", qualifiedByName = "resolveCustomer")
    Order toEntity(CreateOrderRequest request, @Context CustomerRepository customerRepository);

    @Named("resolveCustomer")
    default Customer resolveCustomer(String customerId, @Context CustomerRepository repo) {
        // getReferenceById doesn't hit DB, just creates proxy
        return repo.getReferenceById(customerId);
    }
}
```

### Using Mapper in Service
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;

    public OrderService(OrderRepository orderRepository, OrderMapper orderMapper) {
        this.orderRepository = orderRepository;
        this.orderMapper = orderMapper;
    }

    @Transactional
    public OrderResponse create(CreateOrderCommand command) {
        Order order = Order.create(command.customerId());
        command.items().forEach(item ->
            order.addLine(item.productId(), item.quantity(), item.unitPrice()));
        Order saved = orderRepository.save(order);
        return orderMapper.toResponse(saved);
    }

    @Transactional
    public OrderResponse update(String id, UpdateOrderRequest request) {
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
        orderMapper.updateFromRequest(request, order);
        return orderMapper.toResponse(order);
    }
}
```

## Record Factory Methods

For simple mappings, use static factory methods on records.

### Response DTO with Factory
```java
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
            order.getLines().stream()
                .map(OrderLineResponse::from)
                .toList(),
            order.getTotal().amount(),
            order.getCreatedAt()
        );
    }
}

public record OrderLineResponse(
    String productId,
    int quantity,
    BigDecimal unitPrice,
    BigDecimal subtotal
) {
    public static OrderLineResponse from(OrderLine line) {
        return new OrderLineResponse(
            line.getProductId(),
            line.getQuantity(),
            line.getUnitPrice(),
            line.getSubtotal()
        );
    }
}
```

### Command from Request
```java
public record CreateOrderCommand(
    String customerId,
    List<OrderLineData> items
) {
    public static CreateOrderCommand from(CreateOrderRequest request) {
        return new CreateOrderCommand(
            request.customerId(),
            request.items().stream()
                .map(OrderLineData::from)
                .toList()
        );
    }
}

public record OrderLineData(
    String productId,
    int quantity,
    BigDecimal unitPrice
) {
    public static OrderLineData from(OrderItemRequest request) {
        return new OrderLineData(
            request.productId(),
            request.quantity(),
            request.unitPrice()
        );
    }
}
```

### Using Factory Methods in Service
```java
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public OrderResponse findById(String id) {
        return orderRepository.findById(id)
            .map(OrderResponse::from)  // Method reference to factory
            .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @Transactional(readOnly = true)
    public List<OrderResponse> findByCustomerId(String customerId) {
        return orderRepository.findByCustomerId(customerId).stream()
            .map(OrderResponse::from)
            .toList();
    }
}
```

## Key Rules

### 1. Never Let Clients Set Server-Owned Fields
```java
// Request DTO - only fields client can provide
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemRequest> items
) {
    // NO id, status, createdAt, version fields
}

// Mapper ignores these even if present
@Mapping(target = "id", ignore = true)
@Mapping(target = "status", constant = "PENDING")
@Mapping(target = "createdAt", expression = "java(java.time.Instant.now())")
```

### 2. IDs in DTO, Resolve in Service
```java
// Request contains IDs
public record CreateOrderRequest(
    String customerId,      // Just the ID
    String shippingAddressId // Just the ID
) {}

// Service resolves to entities
@Transactional
public OrderResponse create(CreateOrderRequest request) {
    // Use getReferenceById - doesn't hit DB
    Customer customer = customerRepository.getReferenceById(request.customerId());
    Address address = addressRepository.getReferenceById(request.shippingAddressId());

    Order order = Order.create(customer, address);
    return OrderResponse.from(orderRepository.save(order));
}
```

### 3. Intentional Fetching Before Mapping
```java
// BAD: Mapping triggers N+1 for lines
@Transactional(readOnly = true)
public List<OrderResponse> findAll() {
    return orderRepository.findAll().stream()
        .map(OrderResponse::from) // Each order.getLines() triggers query
        .toList();
}

// GOOD: Fetch eagerly before mapping
@Transactional(readOnly = true)
public List<OrderResponse> findAll() {
    return orderRepository.findAllWithLines().stream() // JOIN FETCH
        .map(OrderResponse::from)
        .toList();
}
```

### 4. Specific DTOs Per Use Case
```java
// List view - minimal fields
public record OrderSummary(
    String id,
    String status,
    BigDecimal total,
    Instant createdAt
) {}

// Detail view - full fields
public record OrderDetail(
    String id,
    String customerId,
    CustomerInfo customer,
    String status,
    List<OrderLineResponse> items,
    BigDecimal total,
    Address shippingAddress,
    Instant createdAt,
    Instant updatedAt
) {}

// Admin view - includes audit fields
public record OrderAdminView(
    String id,
    String status,
    String createdBy,
    String lastModifiedBy,
    List<AuditEvent> auditTrail
) {}
```

### 5. No Mega-DTOs
```java
// BAD: One DTO for all use cases
public record OrderDTO(
    String id,
    String customerId,
    CustomerDTO customer,         // Sometimes null
    String status,
    List<OrderLineDTO> items,     // Sometimes null
    BigDecimal total,
    AddressDTO shippingAddress,   // Sometimes null
    AddressDTO billingAddress,    // Sometimes null
    List<PaymentDTO> payments,    // Sometimes null
    List<ShipmentDTO> shipments,  // Sometimes null
    String createdBy,             // Sometimes null
    Instant createdAt,
    Instant updatedAt
) {}

// GOOD: Purpose-specific DTOs (see #4 above)
```

## Project Structure

```
com.acme.order/
├── application/
│   ├── OrderService.java
│   ├── mapper/
│   │   └── OrderMapper.java
│   └── dto/
│       ├── CreateOrderRequest.java
│       ├── UpdateOrderRequest.java
│       ├── CreateOrderCommand.java
│       ├── OrderResponse.java
│       ├── OrderSummary.java
│       └── OrderLineResponse.java
├── domain/
│   ├── Order.java
│   ├── OrderLine.java
│   └── OrderRepository.java
└── infrastructure/
    ├── JpaOrderRepository.java
    └── OrderController.java
```

### Mapper Location
- Place mappers in `application/mapper/`
- One mapper per aggregate (e.g., `OrderMapper`, `CustomerMapper`)
- Mappers can call other mappers for nested objects
