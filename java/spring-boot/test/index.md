# Testing Strategy Router Index
> Testing Directory Gateway: Navigation map and router delegating execution to specific test type guardrails.

1. Test Type Routing Matrix
When executing or generating tests for Spring Boot 3.x applications, navigate to the relevant test guardrail file based on task intent:

|Target Test Scope|Primary Target / Purpose|File Route
|--|--|--|
|Unit Testing|"Isolated business logic, domain services, helpers, validators, and mappers (No Spring context, strict stubbing enforcement)."|[java/spring-boot/test/unit-testing.md](unit-test.md)|
|Integration Testing|"Full application context execution (@SpringBootTest), Testcontainers for real infrastructure (PostgreSQL), and WireMock for 3rd-party APIs."|[java/spring-boot/test/integration.md](integration-test.md)|