# Project Structure & Directory Structure
> Directory & Package Standards: Guidelines for detecting an existing codebase layout and organizing Spring Boot 3.x applications
This file helps with identification of existing project structure ordeciding new project structure for a new service

## Existing Service Project Structure Identification
Before generating new classes or package trees, the agent MUST inspect the target repository layout.
```text
[Task Received] ──► [Inspect src/main/java/...] ──► [Identify Active Layout Pattern]
                                                               │
                     ┌─────────────────────────────────────────┴─────────────────────────────────────────┐
                     ▼                                                                                   ▼
       [Existing Codebase Found]                                                           [New / Greenfield Project]
                     │                                                                                   │
                     ▼                                                                                   ▼
  [Adopt Existing Pattern: Layer vs Feature]                                             [Enforce Preferred Standard: Feature-First]
```
### Detection Heuristics:

1. Check src/main/java/<base-package>/:
- If top-level directories are named controller, service, repository, adopt Package-by-Layer.
- If top-level directories are domain entities (order, billing, user), adopt Package-by-Feature.

2. Constraint: Never introduce a conflicting package paradigm into an established codebase. Match existing directory naming and placement conventions.

## New Project/Service Project Structure Decision
### Preferred Standard for Greenfield Projects: Hybrid Feature-Layer
When creating new repositories or isolated modules, use a Package-by-Feature top-level structure with internal layer separation:
```text
com.org.appName/
├── Application.java              <-- Application Entrypoint
│
├── order/                        <-- Feature Boundary
│   ├── api/                      <-- REST Controllers & DTOs
│   │   ├── OrderController.java
│   │   └── dto/
│   │       ├── CreateOrderRequest.java
│   │       └── OrderResponse.java
│   ├── domain/                   <-- Business Logic & Entities
│   │   ├── OrderService.java
│   │   └── Order.java
│   └── repository/               <-- Data Access
│       └── OrderRepository.java
│
└── common/                       <-- Cross-Cutting Infrastructure
    ├── config/
    └── security/
```
### Package-by-Layer Standard (Fallback for Legacy Repos)
If the discovery step identifies a Package-by-Layer architecture, place components according to this layout:
```text
com.org.appName/
├── controller/                   <-- All REST Controllers
├── service/                      <-- All Business Services
├── repository/                   <-- All Spring Data Repositories
├── model/                        <-- All JPA Entities
└── dto/                          <-- All Request/Response Records
```

### 4. Class Suffix & Boundary Rules
Regardless of layout pattern:
- Controllers: Must end with Controller (e.g., OrderController).
- Services: Must end with Service (e.g., OrderService).
- Repositories: Must end with Repository (e.g., OrderRepository).
- Cross-Access Constraint: Controllers MUST NOT inject Repositories directly.