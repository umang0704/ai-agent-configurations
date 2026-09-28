# Spring Boot 3.x Guardrails Master Index

> **Spring Boot Master Router**: This index maps package namespaces, architectural layers, and operational domains to modular guardrail files under `java/spring-boot/`.

---

## 1. Stack & Runtime Requirements

* **Target Stack**: Java 21 LTS, Spring Boot 3.x, Spring Data JPA, PostgreSQL.
* **Build System**: Gradle with Kotlin DSL (`build.gradle.kts`) or Groovy DSL (`build.gradle`).
* **Core Libraries**: Lombok (`@RequiredArgsConstructor`, `@Slf4j`), AssertJ, Mockito, Testcontainers.

---

## 2. Package-to-Guardrail Routing Map

Inspect the target package or file path and read **only** the corresponding guardrail file to execute the change:

| Package Pattern / Domain | Focus Area | Target Guardrail File |
| :--- | :--- | :--- |
| `com.org.api.**` | Controllers, HTTP Endpoints, DTOs | [`architecture/api-design.md`](architecture/api-design.md) |
| `com.org.service.**` | Business Logic, Transactions, Domain Mapping | [`architecture/service-design.md`](architecture/service-design.md) |
| `com.org.repository.**` | Spring Data JPA, Entity Mapping | [`architecture/repository-design.md`](architecture/repository-design.md) |
| `db/migration/**`, Entities | DB Schema, Migrations, Indexes | [`architecture/database.md`](architecture/database.md) |
| `com.org.config.security.**` | Spring Security, Auth, Filters | [`architecture/security-design.md`](architecture/security-design.md) |
| Any Logger / Log Config | SLF4J Standards, MDC Context | [`architecture/logging.md`](architecture/logging.md) |
| Package Structure Setup | Project Layout, Layer Rules | [`architecture/project-structure.md`](architecture/project-structure.md) |

---

## 3. Testing & Verification Router

When creating, updating, or debugging tests:

| Test Scope / File Match | Focus Area | Target Guardrail File |
| :--- | :--- | :--- |
| Core Test Rules | Suite Conventions & JUnit 5 Setup | [`test/index.md`](test/index.md) |
| `src/test/**/unit/**` or `*Test.java` | Unit Testing (AssertJ, Mockito) | [`test/unit-test.md`](test/unit-test.md) |
| `src/test/**/integration/**` or `*IT.java` | Testcontainers, MockMvc, DB Integration | [`test/integration-test.md`](test/integration-test.md) |

---

## 4. Build & Operations Router

For build configuration, containerization, or pipeline updates:

| Operation Area | Focus Area | Target Guardrail File |
| :--- | :--- | :--- |
| `build.gradle(.kts)` | Gradle Plugins, Dependencies, Tasks | [`cicd/gradle.md`](cicd/gradle.md) |
| `Dockerfile`, `.dockerignore` | Multi-Stage Builds, Security, OCI | [`cicd/docker.md`](cicd/docker.md) |
| `.github/workflows/**` | CI/CD Pipeline Rules | [`cicd/build.md`](cicd/build.md) |

---

## 5. Execution Workflow for Agents

1. Identify the files or packages involved in the active task using local MCP graph tools (`get_call_graph`, `get_impact_analysis`).
2. Navigate to the specific guardrail file listed above.
3. Apply code modifications adhering to the file's explicit constraints.
4. Verify changes using `./gradlew test` (or relevant test filters).