# REST API Design Guardrails
> API Layer Standards: Technical rules, URI naming conventions, OpenAPI documentation, DTO formats, and reference patterns for constructing REST endpoints in Spring Boot 3.x.

## URI Path & Naming Conventions
- Nouns over Verbs: Endpoints MUST represent resource entities using plural nouns. Never use verbs in paths (e.g., use /api/v1/orders, NEVER /api/v1/getOrders or /api/v1/createOrder).
- Casing: URIs MUST use lowercase kebab-case for multi-word paths (e.g., /api/v1/order-items). Never use camelCase or snake_case in URI paths.
- API Versioning: All paths MUST include a major version prefix (/api/v1/).
- Hierarchical Nesting: Express parent-child relationships through URL paths up to a maximum depth of two levels (e.g., /api/v1/orders/{orderId}/items). Avoid deeper nesting like /api/v1/users/{uId}/orders/{oId}/items/{iId}; flatten deep hierarchies instead.
- Trailing Slashes: URIs MUST NOT end with a trailing slash (/).
- Action Endpoints (RPC Style): For operations that do not map neatly to standard CRUD (e.g., cancelling an order), use a clear verb at the end of the resource path (e.g., POST /api/v1/orders/{id}/cancel).

## Class Annotations: Every controller MUST use @RestController, @RequestMapping("/api/v1/"), @Tag (for Swagger grouping), and @RequiredArgsConstructor.
- Dependency Injection: Use private final fields with Lombok @RequiredArgsConstructor exclusively. Field @Autowired is strictly banned.
- Boundary Isolation: Controllers MUST communicate ONLY with Service components. Direct injection or execution of Spring Data Repositories, JPA Entities, or database components inside a Controller is prohibited.
- Auth Isolation: Controllers MUST NOT contain custom token-parsing or auth-checking logic. Security rules are declared globally in SecurityConfig or via @PreAuthorize annotations.

## HTTP Method & Status Conventions
|Status Code |HTTP Status Text|Category|Use Case / Condition|Response Body Requirement|
|------------|------------|------------|------------|------------|
|200|OK|Success|"Successful GET, PUT, or PATCH request."|Returns updated/retrieved resource DTO.|
|201|Created|Success|Successful POST request (resource created).|Returns created resource DTO + Location header.|
|204|No Content|Success|Successful DELETE request or PUT update with no body response.|Empty response body.|
|400|Bad Request|Client Error|"Payload validation failure (@Valid), syntax error, or unparseable JSON."|RFC 7807 ProblemDetail or standard error response with field errors.|
|401|Unauthorized|Client Error|Missing or invalid authentication token (JWT/OAuth2).|Standard error payload detailing auth requirement.|
|403|Forbidden|Client Error|Authenticated user lacks permission/role for the target resource.|Standard error payload detailing access restriction.|
|404|Not Found|Client Error|Resource with the specified ID does not exist in the database.|Standard error payload detailing missing resource.|
|409|Conflict|Client Error|"Business constraint violation (e.g., duplicate unique field like email)."|Standard error payload detailing conflict reason.|
|422|Unprocessable Entity|Client Error|"Valid syntax, but violates business validation rules."|Standard error payload with specific business rule violations.|
|500|Internal Server Error|Server Error|Uncaught exception or unexpected database failure.|Generic error payload (masks sensitive stack traces).|

## DTO & Validation Rules
- Immutable Records: All Request and Response payloads MUST be defined as Java record types.

- Entity Exposure Constraint: JPA Entities MUST NEVER be accepted in @RequestBody or returned directly from controller methods.

- Validation: Request payloads MUST be annotated with @Valid on the @RequestBody parameter. Apply Jakarta constraints (@NotBlank, @NotNull, @Positive, @Size) directly to record fields.

- Schema Documentation: Use @Schema annotations on record fields to provide example values and human-readable descriptions.

## OpenAPI 3 / Swagger Standards
- Library Mandate: Use org.springdoc:springdoc-openapi-starter-webmvc-ui (Spring Boot 3 / Jakarta compatible). Legacy springfox or io.swagger v1/v2 annotations are banned.

- Class Tagging: Annotate the Controller class with @Tag(name = "", description = "").

- Method Operations: Annotate every endpoint with @Operation(summary = "...", description = "...").

- Responses: Explicitly declare standard success and error response codes using @ApiResponses and @ApiResponse.

## Canonical Controller Pattern
```java
package com.org.appName.order.api;

import com.org.appName.order.api.dto.CreateOrderRequest;
import com.org.appName.order.api.dto.OrderResponse;
import com.org.appName.order.domain.OrderService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@Tag(name = "Orders", description = "Endpoints for managing customer order lifecycles")
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

@Operation(summary = "Create a new order", description = "Validates the request payload and creates an order entity.")
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "Order created successfully"),
    @ApiResponse(responseCode = "400", description = "Invalid request payload")
})
@PostMapping
public ResponseEntity<OrderResponse> createOrder(@RequestBody @Valid CreateOrderRequest request) {
    OrderResponse response = orderService.createOrder(request);
    URI location = URI.create("/api/v1/orders/" + response.id());
    return ResponseEntity.created(location).body(response);
}

@Operation(summary = "Get order by ID")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Order found"),
    @ApiResponse(responseCode = "404", description = "Order not found")
})
@GetMapping("/{id}")
public ResponseEntity<OrderResponse> getOrderById(@PathVariable Long id) {
    OrderResponse response = orderService.getOrderById(id);
    return ResponseEntity.ok(response);
}

@Operation(summary = "List all orders", description = "Returns a paginated list of orders.")
@GetMapping
public ResponseEntity<Page<OrderResponse>> getAllOrders(Pageable pageable) {
    Page<OrderResponse> responses = orderService.getAllOrders(pageable);
    return ResponseEntity.ok(responses);
}

@Operation(summary = "Cancel an active order")
@PostMapping("/{id}/cancel")
public ResponseEntity<OrderResponse> cancelOrder(@PathVariable Long id) {
    OrderResponse response = orderService.cancelOrder(id);
    return ResponseEntity.ok(response);
}

@Operation(summary = "Delete an order")
@ApiResponse(responseCode = "204", description = "Order successfully deleted")
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
    orderService.deleteOrder(id);
    return ResponseEntity.noContent().build();
}
}
```

### Canonical DTO Record Pattern
```java
package com.org.appName.order.api.dto;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import java.math.BigDecimal;

@Schema(description = "Request payload for creating a new order")
public record CreateOrderRequest(
@Schema(description = "Unique identifier of the customer", example = "CUST-1002")
@NotBlank(message = "Customer ID is required")
String customerId,

@Schema(description = "Total order amount in USD", example = "299.99")
@NotNull(message = "Amount is required")
@Positive(message = "Amount must be greater than zero")
BigDecimal amount
) {}

@Schema(description = "Response payload representing an order")
public record OrderResponse(
@Schema(description = "Primary key ID", example = "101")
Long id,

@Schema(description = "Unique identifier of the customer", example = "CUST-1002")
String customerId,

@Schema(description = "Total order amount", example = "299.99")
BigDecimal amount,

@Schema(description = "Current lifecycle status", example = "CREATED")
String status
) {}
```