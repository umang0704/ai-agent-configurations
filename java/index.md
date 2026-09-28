# Java Language Framework Gateway
>Primary Router: Language sub-index routing execution tasks to targeted Java framework guardrails.

1. Framework Delegation Matrix

When evaluating or generating Java backend code, route execution based on the target framework detected in build descriptors (pom.xml / build.gradle):

|Target Framework|Detection Signatures|Primary Router File Path|
|Spring Boot 3.x|"org.springframework.boot, @SpringBootApplication"|[java/spring-boot/index.md](spring-boot/index.md)|
|Dropwizard 4.x|"io.dropwizard, io.dropwizard.core.Application"|[java/dropwizard/index.md](dropwizard/index.md)|