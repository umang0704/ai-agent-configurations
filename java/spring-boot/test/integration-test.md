# Integration Testing Guardrails
> Integration Testing Standards: Application Context boots, Testcontainers, WireMock 3rd-party API mocking, end-to-end API behaviour validation, and database state management for Spring Boot 3.x.

## 1. Full Application Context & Environment Setup
1. Full Context Boot: Integration tests MUST boot the complete Spring Application Context using @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT).

2. Real Infrastructure First: Databases (PostgreSQL) and cache layers (Redis) MUST run against real instances using Testcontainers. Embedded in-memory DBs (like H2) are prohibited for integration tests to ensure production SQL dialect compatibility.

3. Third-Party API Mocking Policy: External 3rd-party HTTP services (e.g., payment gateways, external auth servers) MUST NOT be hit in tests. Mock external APIs using WireMock or @ServiceConnection HTTP mocks. Do NOT use @MockBean for internal services/repositories in integration tests—let the real internal beans wire together end-to-end.


## 2. End-to-End API Behavior & Validation Rules
- HTTP Client Standard: Use TestRestTemplate or WebTestClient to send real HTTP requests to the running random server port.

- Detailed Behavior Coverage: Integration tests MUST validate the full lifecycle behavior of API endpoints:

    - HTTP Response Status Codes & Headers (Location, Content-Type).

    - Response Payload Contract (Fields, nested JSON structure).

    - Database State Mutations (Verify records are actually created, updated, or removed in the DB post-request).

    - Error Behaviors & ProblemDetails RFC 7807 responses on invalid input or constraint violations.

- Database Isolation: Tests MUST remain independent and repeatable. Clean up database state between tests using @Transactional, @Sql, or an explicit cleanup utility method.

## 3. Strict Stubbing Policy & Lenient Ban
- Lenient Stubbing Ban: @MockitoSettings(strictness = Strictness.LENIENT) or lenient().when(...) is STRICTLY BANNED. Every defined stub MUST be invoked during test execution. Unnecessary or unused stubs indicate dead code and false test confidence.

- UnnecessaryStubbingException Protocol: All tests must run with Mockito's default Strict Stubs setting (Strictness.STRICT_STUBS). Tests MUST fail if an unused stub is declared.

- Specific Argument Matchers over Any: Avoid blanket any() matchers in stubs unless argument values are truly dynamic. Prefer exact values or strict matchers (eq(), argThat()) to ensure mock expectations are exact.

- Minimal Stubbing Principle: Only stub methods that are directly called during the specific test execution path. If a branch does not invoke a dependency, do not write a given(...) or when(...) for it.

3. Canonical Integration Test Pattern
```java
package com.org.appName.order.domain;

import com.org.appName.order.api.dto.CreateOrderRequest;
import com.org.appName.order.api.dto.OrderResponse;
import com.org.appName.order.repository.OrderRepository;
import com.org.appName.common.exception.BusinessValidationException;
import com.org.appName.common.exception.ResourceNotFoundException;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.mockito.junit.jupiter.MockitoSettings;
import org.mockito.quality.Strictness;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.STRICT_STUBS)
@DisplayName("OrderService Unit Tests (Strict Stubbing)")
class OrderServiceImplTest {

@Mock
private OrderRepository orderRepository;

@Mock
private OrderValidator orderValidator;

@Mock
private OrderMapper orderMapper;

@InjectMocks
private OrderServiceImpl orderService;

@Nested
@DisplayName("Create Order Logic")
class CreateOrder {

    @Test
    @DisplayName("Should successfully create order when request payload is valid")
    void createOrder_Success() {
        // Given: Only stub methods that WILL be executed
        CreateOrderRequest request = new CreateOrderRequest("CUST-100", new BigDecimal("150.00"));
        Order order = Order.builder().id(1L).customerId("CUST-100").amount(new BigDecimal("150.00")).status("CREATED").build();
        OrderResponse expectedResponse = new OrderResponse(1L, "CUST-100", new BigDecimal("150.00"), "CREATED");

        given(orderMapper.toEntity(request)).willReturn(order);
        given(orderRepository.save(any(Order.class))).willReturn(order);
        given(orderMapper.toResponse(order)).willReturn(expectedResponse);

        // When
        OrderResponse actualResponse = orderService.createOrder(request);

        // Then
        assertThat(actualResponse).isNotNull();
        assertThat(actualResponse.id()).isEqualTo(1L);
        assertThat(actualResponse.customerId()).isEqualTo("CUST-100");
        verify(orderValidator).validateOrderCreation(request);
        verify(orderRepository).save(order);
    }

    @ParameterizedTest(name = "Should throw BusinessValidationException for invalid amount: {0}")
    @ValueSource(strings = {"0.00", "-10.00", "-500.50"})
    @DisplayName("Should reject order creation when amount is non-positive")
    void createOrder_InvalidAmount_ThrowsException(String invalidAmount) {
        // Given: Only stub the validator; DO NOT stub orderMapper or orderRepository because execution short-circuits
        CreateOrderRequest request = new CreateOrderRequest("CUST-100", new BigDecimal(invalidAmount));

        given(orderValidator.validateOrderCreation(request))
                .willThrow(new BusinessValidationException("Amount must be greater than zero"));

        // When / Then
        assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(BusinessValidationException.class)
                .hasMessageContaining("Amount must be greater than zero");

        // Verify no downstream interactions occurred
        verifyNoInteractions(orderMapper, orderRepository);
    }
}

@Nested
@DisplayName("Get & Update Order Logic")
class GetAndUpdateOrder {

    @Test
    @DisplayName("Should return order response when valid ID exists")
    void getOrderById_Found() {
        // Given
        Long orderId = 1L;
        Order order = Order.builder().id(orderId).customerId("CUST-100").amount(new BigDecimal("100.00")).status("CREATED").build();
        OrderResponse expectedResponse = new OrderResponse(orderId, "CUST-100", new BigDecimal("100.00"), "CREATED");

        given(orderRepository.findById(orderId)).willReturn(Optional.of(order));
        given(orderMapper.toResponse(order)).willReturn(expectedResponse);

        // When
        OrderResponse response = orderService.getOrderById(orderId);

        // Then
        assertThat(response).isNotNull();
        assertThat(response.id()).isEqualTo(orderId);
    }

    @ParameterizedTest(name = "Should throw ResourceNotFoundException for non-existent ID: {0}")
    @ValueSource(longs = {99L, 999L, 5000L})
    @DisplayName("Should throw ResourceNotFoundException when order ID is missing")
    void getOrderById_NotFound_ThrowsException(Long nonExistentId) {
        // Given: Stub repository returning empty Optional; DO NOT stub orderMapper
        given(orderRepository.findById(nonExistentId)).willReturn(Optional.empty());

        // When / Then
        assertThatThrownBy(() -> orderService.getOrderById(nonExistentId))
                .isInstanceOf(ResourceNotFoundException.class)
                .hasMessageContaining("Order not found with id: " + nonExistentId);

        verifyNoInteractions(orderMapper);
    }
}
}
```