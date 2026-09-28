# Service Layer Design Guardrails
>Service Layer Standards: Interface-based contract design, size limits, transaction management, exception strategy, and progressive helper decomposition for Spring Boot 3.x.

## 1. Interface-Driven Service Design
- Contract Isolation: Every service MUST define a public Java interface (e.g., OrderService) exposing domain operations, accompanied by a package-private implementation class (e.g., OrderServiceImpl).

- Dependency Type: Controllers and external components MUST inject the interface type (OrderService), never the concrete implementation class directly.

- Encapsulation: Keep concrete implementation classes package-private when using feature-first packaging to prevent direct instantiation across feature boundaries.

## 2. Service Sizing & Progressive Helper Decomposition
- Class Size Cap: A service implementation class MUST NOT exceed 2000 lines of code.

- Progressive Extraction Rule:

    - Small/Low-Complexity Features: Supporting private helper methods may remain within the service class or a single package-private OrderServiceHelper.

    - Growing Complexity: As logic broadens or file length approaches the threshold, extract specialized components:

    - Mappers (OrderMapper): Dedicated to converting between Entities, DTOs, and internal models.

    - Validators (OrderValidator): Dedicated to complex state or multi-field business rules.

    - Clients / Integrations (PaymentGatewayClient): Dedicated to external HTTP or RPC interactions.

- Single Responsibility: The core service class handles workflow orchestration, transaction boundaries, and domain events—delegating boilerplate transformation and validation to decomposed helpers.

## 3. Transaction Management & Null Safety
- Class-Level Defaults: Annotate service implementations with @Transactional(readOnly = true) to make all read operations non-locking by default.

- Write Operations: Explicitly annotate mutating methods (create, update, delete) with @Transactional.

- Null Safety: Single object lookup methods MUST return java.util.Optional<T> or throw a custom domain exception (e.g., ResourceNotFoundException) directly. Never return bare null.

## 4. Canonical Service & Helper Decomposition Pattern
```java
package com.org.appName.order.domain;

import com.org.appName.order.api.dto.CreateOrderRequest;
import com.org.appName.order.api.dto.OrderResponse;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface OrderService {
OrderResponse createOrder(CreateOrderRequest request);
OrderResponse getOrderById(Long id);
Page getAllOrders(Pageable pageable);
OrderResponse cancelOrder(Long id);
}
```

```java
package com.org.appName.order.domain;

import com.org.appName.order.api.dto.CreateOrderRequest;
import com.org.appName.order.api.dto.OrderResponse;
import com.org.appName.order.repository.OrderRepository;
import com.org.appName.common.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
class OrderServiceImpl implements OrderService {

private final OrderRepository orderRepository;
private final OrderValidator orderValidator;
private final OrderMapper orderMapper;

@Override
@Transactional
public OrderResponse createOrder(CreateOrderRequest request) {
    log.info("Processing order creation for customer: {}", request.customerId());

    // Delegate validation to dedicated component
    orderValidator.validateOrderCreation(request);

    // Delegate mapping to entity
    Order order = orderMapper.toEntity(request);
    order.setStatus("CREATED");

    Order savedOrder = orderRepository.save(order);
    return orderMapper.toResponse(savedOrder);
}

@Override
public OrderResponse getOrderById(Long id) {
    return orderRepository.findById(id)
            .map(orderMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found with id: " + id));
}

@Override
public Page<OrderResponse> getAllOrders(Pageable pageable) {
    return orderRepository.findAll(pageable)
            .map(orderMapper::toResponse);
}

@Override
@Transactional
public OrderResponse cancelOrder(Long id) {
    Order order = orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found with id: " + id));

    orderValidator.validateOrderCancellation(order);
    order.setStatus("CANCELLED");

    Order updatedOrder = orderRepository.save(order);
    return orderMapper.toResponse(updatedOrder);
}
}
```

```java
package com.org.appName.order.domain;

import com.org.appName.order.api.dto.CreateOrderRequest;
import com.org.appName.order.api.dto.OrderResponse;
import com.org.appName.common.exception.BusinessValidationException;
import org.springframework.stereotype.Component;

@Component
class OrderValidator {

public void validateOrderCreation(CreateOrderRequest request) {
    if (request.amount() == null || request.amount().signum() <= 0) {
        throw new BusinessValidationException("Order amount must be greater than zero");
    }
}

public void validateOrderCancellation(Order order) {
    if ("COMPLETED".equals(order.getStatus())) {
        throw new BusinessValidationException("Completed orders cannot be cancelled");
    }
}
}

@Component
class OrderMapper {

public Order toEntity(CreateOrderRequest request) {
    return Order.builder()
            .customerId(request.customerId())
            .amount(request.amount())
            .build();
}

public OrderResponse toResponse(Order order) {
    return new OrderResponse(
            order.getId(),
            order.getCustomerId(),
            order.getAmount(),
            order.getStatus()
    );
}
}
```