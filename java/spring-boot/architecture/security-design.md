# Security Architecture Guardrails (Extensible Standard)
> Security Guardrails: Core principles, zero-trust rules, and framework-specific adapter patterns for application security, M2M workloads, and identity propagation.

## 1. Universal Security Principles (Framework-Agnostic)
- Stateless Authenticated Requests: Services MUST be stateless. Every incoming request must carry cryptographically verifiable credentials (OAuth2 Bearer JWT, mTLS X.509 SVID, or API Keys).

- Earliest Execution Boundary: Authentication and transport security checks MUST run at the ingress gateway or servlet/HTTP container filter layer—BEFORE reaching application handlers or controllers.

- Domain Identity Abstraction: Application business logic MUST NEVER read raw headers, tokens, or X.509 certificates directly. Identity details MUST be mapped into a standardized SecurityPrincipal contract:

    - UserPrincipal (human user identity, email, roles, scopes)

    - WorkloadPrincipal (M2M / SPIFFE SVID, e.g., spiffe://<domain>/ns/<ns>/sa/<service>)

- Least Privilege Authorization: Enforce attribute-based (ABAC) or role-based (RBAC) authorization declaratively at both route and domain method boundaries.

- RFC 7807 Standard Error Responses: Unauthenticated (401) or Forbidden (403) access attempts MUST return standardized RFC 7807 error responses without leaking internal stack traces.

## 2. Pluggable Identity & Auth Matrix
|Credential Type | Provider / Trust Domain | Target Identity Model | Resolution Mechanism
|--|--|--|--|
| OAuth2 / OIDC JWT|Keycloak, Okta, Auth0 | UserPrincipal | JWKS Endpoint Signature & Claim Validation |
| JWT-SVID (Bearer)|SPIFFE / SPIRE Agent | WorkloadPrincipal | SPIRE OIDC Discovery & Audience Check |
| X.509 SVID (mTLS)|SPIFFE / SPIRE CA / Vault | WorkloadPrincipal | SAN URI Extraction (spiffe://...) |
| API Key / Secret|Internal Secret Manager | ServicePrincipal | Hash Lookup & Rate Limit Check |

## 3. Spring Boot Security Adapter (java/spring-boot/)
When generating code for Spring Boot 3.x applications, implement the security abstractions using native Spring Security 6.x capabilities:

- Security Filter Chain: Use SecurityFilterChain bean definitions with sessionCreationPolicy(SessionCreationPolicy.STATELESS).

- OAuth2 Resource Server: Use .oauth2ResourceServer(oauth2 -> oauth2.jwt(...)) with a custom Converter<Jwt, AbstractAuthenticationToken> to map claims into GrantedAuthority collections.

- Method Security: Enable @EnableMethodSecurity and use @PreAuthorize("hasAuthority(...)") on service methods.

- Identity Injection: Inject principals into controllers using @AuthenticationPrincipal.

## 4. Alternative Framework Extensions (Future Adapters)
- Dropwizard / Jakarta EE: Implement security via ContainerRequestFilter (Jersey filters) and @Auth annotation bindings.

- Go (Gin / Chi): Implement security via custom HTTP Middleware populating context.Context.

- Python (FastAPI): Implement security via FastAPI Depends() security dependencies and OAuth2 HTTP Bearer schemes.

## Canonical Spring Boot Security Pattern
```java
package com.org.appName.common.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/v1/public/**", "/v3/api-docs/**", "/swagger-ui/**").permitAll()
            .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}));

    return http.build();
}
}
```