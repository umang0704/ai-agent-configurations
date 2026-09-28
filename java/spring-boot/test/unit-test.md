# Unit Testing Guardrails

> Unit Testing Standards: JUnit 5 annotations, Mockito initialization, private method testing policies, parameterized tests, and logical branch coverage for Spring Boot 3.x.

## 1. Core Framework & Annotation Standards
- Framework Mandate: Use JUnit 5 (org.junit.jupiter.api.*) and AssertJ (org.assertj.core.api.Assertions) exclusively. JUnit 4 is strictly banned.

- Spring Boot & Mockito Extension: Unit tests MUST run without booting the Spring application context. Use @ExtendWith(MockitoExtension.class) on the test class.

- Injection Pattern: Use Mockito's @InjectMocks to instantiate the class under test, and @Mock (or @Spy) for all collaborators. Do NOT use @Autowired or @MockBean in pure unit tests.

- Human-Readable Documentation: Every test method MUST carry a descriptive @DisplayName("...") annotation clearly stating the scenario and expected outcome.

## 2. Encapsulation & Private Method Policy
- No Direct Reflection by Default: Private methods MUST NOT be tested directly using reflection (e.g., ReflectionTestUtils or setAccessible(true)). Private functions are implementation details and MUST be tested transitively through public class methods.

- Reflection Strict Exception: Reflection or package-private visibility overrides for testing are allowed ONLY AND ONLY IF a critical safety path cannot be triggered through public APIs (e.g., legacy code refactoring safeguards or untestable low-level lifecycle hooks) and requires explicit engineering justification.

## 3. Logical Branching & Parameterized Tests
- Granular Branch Coverage: Write individual or parameterized test cases for every distinct logical component and branch within a method (e.g., success paths, validation failures, boundary values, and exception handling).

- Parameterized Testing Mandate: Whenever a method contains multiple input combinations or edge cases following the same execution path, use JUnit 5 parameterized tests (@ParameterizedTest) with sources like @ValueSource, @CsvSource, or @MethodSource instead of duplicating test methods.

- BDD Style Structure: Organize test method bodies cleanly into Given-When-Then (Arrange-Act-Assert) sections.

## 4. Canonical Unit Test Pattern
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

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.verifyNoInteractions;

@ExtendWith(MockitoExtension.class)
@DisplayName("OrderService Unit Tests")
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
        // Given
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
        // Given
        CreateOrderRequest request = new CreateOrderRequest("CUST-100", new BigDecimal(invalidAmount));

        // Simulate validator throwing exception for invalid amounts
        given(orderValidator.validateOrderCreation(request))
                .willThrow(new BusinessValidationException("Amount must be greater than zero"));

        // When / Then
        assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(BusinessValidationException.class)
                .hasMessageContaining("Amount must be greater than zero");

        verifyNoInteractions(orderMapper);
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
        // Given
        given(orderRepository.findById(nonExistentId)).willReturn(Optional.empty());

        // When / Then
        assertThatThrownBy(() -> orderService.getOrderById(nonExistentId))
                .isInstanceOf(ResourceNotFoundException.class)
                .hasMessageContaining("Order not found with id: " + nonExistentId);
    }

    @ParameterizedTest(name = "Order status transition check - Status: {0}, Allowed: {1}")
    @CsvSource({
        "CREATED, true",
        "PENDING, true",
        "COMPLETED, false",
        "CANCELLED, false"
    })
    @DisplayName("Should validate cancellation eligibility based on order status")
    void cancelOrder_StatusCheck(String initialStatus, boolean isAllowed) {
        // Given
        Long orderId = 10L;
        Order order = Order.builder().id(orderId).status(initialStatus).build();

        given(orderRepository.findById(orderId)).willReturn(Optional.of(order));

        if (!isAllowed) {
            given(orderValidator.validateOrderCancellation(order))
                    .willThrow(new BusinessValidationException("Order cannot be cancelled in state: " + initialStatus));

            // When / Then
            assertThatThrownBy(() -> orderService.cancelOrder(orderId))
                    .isInstanceOf(BusinessValidationException.class);
        } else {
            Order updatedOrder = Order.builder().id(orderId).status("CANCELLED").build();
            OrderResponse expectedResponse = new OrderResponse(orderId, "CUST-100", new BigDecimal("100.00"), "CANCELLED");

            given(orderRepository.save(order)).willReturn(updatedOrder);
            given(orderMapper.toResponse(updatedOrder)).willReturn(expectedResponse);

            // When
            OrderResponse response = orderService.cancelOrder(orderId);

            // Then
            assertThat(response.status()).isEqualTo("CANCELLED");
        }
    }
}
}
```