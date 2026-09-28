# Database & Schema Guardrails (SQL & NoSQL)
>Database Layer Standards: Migration rules, schema design conventions, transaction boundaries, and indexing policies for SQL and NoSQL stores in Spring Boot 3.x.
## 1. Store Selection & Polyglot Boundaries
- Relational Core (PostgreSQL): Use for transactional domain models, financial data, and strong consistency requirements (ACID).
- Document Store (MongoDB): Use for unstructured payloads, audit logs, catalog items, and dynamic event attributes.
- Key-Value Cache (Redis): Use for distributed session state, temporary caching, and fast volatile storage.
- Annotation Separation Constraint: NEVER combine SQL annotations (@Entity, @Table) with NoSQL annotations (@Document, @RedisHash) on the same Java class. Keep persistence models completely isolated.

## 2. SQL Standards (PostgreSQL)
1. Migration Tool Mandate: All relational schema changes MUST be executed via versioned Flyway migrations located in src/main/resources/db/migration/.
2. Naming Conventions:
- Tables and columns MUST use lowercase snake_case (e.g., orders, customer_id).
- Table names MUST be plural (e.g., orders, payments).
- Primary keys MUST be named id.
- Foreign keys MUST use the format <referenced_singular_table>_id (e.g., order_id).

3. Database Type Mapping:
- Monetary & Precise Decimals: Use DECIMAL/NUMERIC. Never use FLOAT or DOUBLE.
- Timestamps: Always use TIMESTAMP WITH TIME ZONE (TIMESTAMPTZ).
- Identifiers: Prefer BIGINT or UUID over 32-bit INT.

4. Index & Constraint Policy:
- Foreign key columns MUST have explicit database-level constraints and indexes.
- Every table MUST include audit columns: created_at and updated_at.

## 3. NoSQL Standards (MongoDB & Redis)
- MongoDB Collection Naming: Collection names MUST use lowercase snake_case and plural form (e.g., audit_logs).
- Document Identifiers: Primary key fields in document models MUST be named id and annotated with @Id.
- TTL & Memory Management (Redis):
    - Every Redis entity or cached key MUST have an explicit Time-To-Live (TTL) set (e.g., @RedisHash(timeToLive = ...)).
    - Never store unbounded collections in Redis without a eviction policy.
- Index Declarations:
    - Annotate frequently queried document fields with @Indexed or create Compound Indexes via @CompoundIndex.

## 4. Transaction & Consistency Rules
- Single-Store Transactions: Use @Transactional exclusively for operations within the SQL database.
- Cross-Store Consistency: Do NOT attempt to run a single @Transactional boundary across both PostgreSQL and MongoDB/Redis. Use asynchronous events or transactional outbox patterns for eventual consistency.

## 5. Canonical SQL Migration Pattern (Flyway)
```sql
CREATE TABLE orders (
id BIGSERIAL PRIMARY KEY,
customer_id VARCHAR(64) NOT NULL,
amount NUMERIC(12, 2) NOT NULL,
status VARCHAR(32) NOT NULL,
created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

## 6. Canonical Entity & Document Patterns

```java
// --- PostgreSQL Entity Pattern ---
package com.org.appName.order.domain;

import jakarta.persistence.;
import lombok.;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Column(name = "customer_id", nullable = false)
private String customerId;

@Column(name = "amount", nullable = false, precision = 12, scale = 2)
private BigDecimal amount;

@Column(name = "status", nullable = false)
private String status;

@Column(name = "created_at", nullable = false, updatable = false)
private Instant createdAt;
}

// --- MongoDB Document Pattern ---
package com.org.appName.audit.domain;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;
import java.time.Instant;

@Document(collection = "audit_logs")
@Getter
@Builder
public class AuditLog {

@Id
private String id;

@Indexed
private String userId;

private String action;
private Instant timestamp;
}
```