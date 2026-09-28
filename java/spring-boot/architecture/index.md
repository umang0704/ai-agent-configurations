# Spring Boot Architecture Router Index
| Target Scope | Responsible Module | File Path |
|--|--|--|
| Package Structure & Discovery | "Layout  packaging rules  & discovery" | [java/spring-boot/architecture/project-structure.md](project-structure.md) |
| REST Controllers & DTOs | "Request mapping  validation OpenAPI" | [java/spring-boot/architecture/api-design.md](api-design.md) |
| Business Logic & Domain | "Services  interface contracts  helpers" | [java/spring-boot/architecture/service-design.md](service-design.md) |
| Data Access & Repositories | Spring Data interfaces & JPA queries | [java/spring-boot/architecture/repository-design.md](repository-design.md) |
| Security & Identity | "Ingress filters RBAC  M2M abstractions" | [java/spring-boot/architecture/security-design.md](security-design.md) |
| Observability & Logging | "SLF4J MDC tracing & structured JSON" | [java/spring-boot/architecture/logging.md](logging.md) |

## 2. Core Execution Constraints
- Strict Layer Isolation:
    - Controller $\rightarrow$ Service $\rightarrow$ Repository.
    - Controllers MUST NOT inject repositories or database entities directly.

- Immutability Standard:
    - DTOs MUST use Java record types exclusively.

- Dependency Injection Standard:
    - Constructor injection via Lombok @RequiredArgsConstructor exclusively; field @Autowired is banned.
- Service Size & Decomposition:
    - Service implementation classes MUST NOT exceed 2000 lines.
    - Progressive extraction MUST delegate to dedicated Validator, Mapper, and Client components as complexity grows.

```text
java/spring-boot/architecture/
|-- index.md                   <-- Router Gateway (This File)
|-- project-structure.md      <-- Packaging & Discovery
|-- api-design.md             <-- REST Endpoints, DTOs & OpenAPI
|-- service-design.md         <-- Interfaces, Services & Helpers
|-- repository-design.md      <-- Data Access & Repositories
|-- security-design.md        <-- Stateless Security Filters & Identity
+-- logging.md                <-- SLF4J, MDC Tracing & Observability
```