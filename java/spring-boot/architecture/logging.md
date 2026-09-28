# Logging Guardrails
> Logging Standards: Guidelines for structured logging, industry-standard observability integration (OpenTelemetry/Micrometer), MDC contextual tracing, log levels, and reference patterns in Spring Boot 3.x.

## 1. Core Logging Principles
- Abstraction Mandate: Use the SLF4J facade via Lombok @Slf4j exclusively. Direct references to System.out, System.err, java.util.logging, or Log4j2 classes are strictly prohibited.

- Structured Output: In non-local environments (staging, production), output logs in structured JSON format (e.g., using Logback's logstash-logback-encoder) to enable log aggregation platforms (Datadog, Elastic, CloudWatch) to parse fields automatically.

- Parameterization over String Concatenation: Use SLF4J parameterized placeholders (e.g., log.info("Processing order for customerId: {}", customerId)) instead of string concatenation (+) to prevent unnecessary String object allocation when log levels are disabled.

## 2. Contextual Tracing (MDC)
- To trace requests across microservice boundaries, log statements must carry diagnostic context:

- Correlation IDs: Interceptors/Filters MUST populate the Mapped Diagnostic Context (MDC) with a traceId (or correlationId) and spanId upon receiving an HTTP request.

- MDC Cleanup: Custom filters adding items to MDC MUST clear the context in a finally block or use a try-with-resources block to prevent MDC context leak across reused thread pool threads.

## 3. Log Level Guidelines
|Log Level |Purpose / Operational Trigger|Example Use Case|
|--|--|--|
ERROR | Application error that requires immediate engineering/ops attention or indicates a degraded system component.|Database connection pool exhaustion, unhandled 500 exceptions, payment gateway connection failures.|
WARN | Unexpected condition or recovered failure that does not halt request execution, but warrants tracking.|Transient retry attempts, deprecated API endpoint usage, fallback cache reads on DB timeout.|
INFO | Key state changes, business milestones, and lifecycle events. Keep concise to avoid log flooding.|Application startup, order status updates (CREATED $\rightarrow$ PAID), scheduled batch job completions.|
DEBUG | Detailed diagnostic information useful during active development or troubleshooting.|Input/output payload structures, internal state checks, query filter evaluation.TRACEHighly verbose step-by-step framework internals. Banned in non-development environments.|Raw byte stream decoding, internal security filter evaluation loops.|

## 4. Sensitive Data & Security Masking
- PII & Credential Shielding: Raw passwords, authorization tokens, bearer tokens, credit card numbers, social security numbers, and secret keys MUST NEVER be logged.

- Masking Encoders: Configure custom Logback pattern layout masks or Jackson sanitizers to automatically redact sensitive JSON keys (e.g., "password": "***", "token": "***").

## 5. Canonical Logging & MDC Pattern
```java
package com.org.appName.common.logging;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(1)
public class TraceLoggingFilter extends OncePerRequestFilter {

private static final String TRACE_HEADER = "X-Trace-Id";
private static final String TRACE_MDC_KEY = "traceId";

@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain filterChain) throws ServletException, IOException {
    String traceId = request.getHeader(TRACE_HEADER);
    if (traceId == null || traceId.isBlank()) {
        traceId = UUID.randomUUID().toString();
    }

    MDC.put(TRACE_MDC_KEY, traceId);
    response.setHeader(TRACE_HEADER, traceId);

    try {
        filterChain.doFilter(request, response);
    } finally {
        MDC.remove(TRACE_MDC_KEY);
    }
}
}
```

## 6. Canonical Service Logging Usage
```java
package com.org.appName.order.domain;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderProcessingService {

public void processOrder(String orderId, double amount) {
    log.info("Starting processing for orderId: {} with amount: {}", orderId, amount);

    try {
        // Business logic execution
        log.debug("Evaluating fraud checks for orderId: {}", orderId);
    } catch (Exception ex) {
        log.error("Failed to process orderId: {}. Error: {}", orderId, ex.getMessage(), ex);
        throw ex;
    }
}
}
```