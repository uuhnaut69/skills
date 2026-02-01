---
name: java-spring
description: Best practices for Java/Spring projects. Covers DDD principles, explicit code (no Lombok), Spring Boot 4 features, REST APIs, testing with Testcontainers, and observability.
---

# Java Spring Development Guide

## Core Principles

### 1. Domain-Driven Design (DDD)

Structure code around business capabilities, not technical layers:

```
com.acme.app/
├── billing/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
├── ordering/
└── shipping/
```

### 2. No Lombok

Write explicit Java code—no `@Data`, `@Builder`, `@Getter`:

- Use Java records for immutable data (DTOs, value objects)
- Use IDE generation for entity getters/setters
- Explicit code is easier to debug and understand

### 3. Conservative Dependency Management

Check existing dependencies before adding new ones:

```bash
./mvnw dependency:tree | grep -E "(spring|test)"
```

Spring Boot starters include many transitive dependencies.

### 4. Constructor Injection

Always use constructor injection, never field injection:

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

## Quick Reference

| Scenario                | Action                                         |
| ----------------------- | ---------------------------------------------- |
| Need a DTO              | Use Java record                                |
| Need entity             | Write class with explicit getters, `@Version`  |
| Need builder            | Use record or static inner `Builder` class     |
| Need JSON mapping       | Use Jackson annotations on records             |
| Need database           | Use Spring Data JPA repository                 |
| Need DTO↔Entity         | MapStruct (complex) or record factory (simple) |
| Controller needs entity | NO—return DTO from service                     |
| Service modifies data   | Add `@Transactional` on public method          |
| External HTTP call      | HTTP Interface Client or RestClient            |

## Reference Documents

### When to load additional references:

**Load `references/ddd-patterns.md` when:**

- Designing new modules or bounded contexts
- Working with aggregates, entities, value objects
- Implementing domain events
- Setting up package structure

**Load `references/data-access.md` when:**

- Creating JPA entities (mapping, relationships)
- Preventing N+1 queries
- Implementing optimistic/pessimistic locking
- Setting up auditing or Flyway migrations

**Load `references/transactions.md` when:**

- Defining transaction boundaries
- Understanding propagation modes
- Handling rollback scenarios
- Debugging transaction issues (self-invocation, lazy loading)

**Load `references/testing.md` when:**

- Setting up Testcontainers
- Writing integration tests
- Choosing test slices (`@WebMvcTest`, `@DataJpaTest`)
- Optimizing test performance

**Load `references/mapping.md` when:**

- Mapping between DTOs and entities
- Setting up MapStruct
- Designing request/response DTOs
- Understanding layer responsibilities

**Load `references/spring-boot-best-practices.md` when:**

- Designing REST APIs (controllers, status codes)
- Implementing error handling with ProblemDetail
- Configuring `@ConfigurationProperties`
- Setting up observability (Micrometer, `@Observed`)
- Enabling structured logging

**Load `references/spring-boot-4.md` when:**

- Working with Spring Boot 4.x projects
- Migrating to Jackson 3 / JsonMapper
- Using JSpecify null safety
- Implementing `@Retryable`, `@ConcurrencyLimit`
- Using RestTestClient
- Registering HTTP clients with `@ImportHttpServices`
