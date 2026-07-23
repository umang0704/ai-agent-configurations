# Java Spring Boot Conventions

## Security Configuration
- Security Configurations should be created with optimised filters
## Filters
- Each filter created should have a unique responsibility
- Early 'if' patterns should be used to skip filters
## Controller Layer
- controller layers should only have service dependency
- controller layers should only have business validations
- controller layer classes should not exceed 1000 lines of code, appropriate breakdown of architecture should happen

## Service Layers
- service layers should be defined for the use cases
  - entity/resources services
  - operation service : validation, mapper, static utils etc
  - feature services : services which handles interactions between 2 resource services for a feature
- Service layers should have dependency for database layer, helper service layers, operation layers
- Service layers classes should not exceed more thant 1500 lines of code
- Service layers should be created in interface and implementation class
- Service layers interfaces should have java strings

### Helper Layers
- Helper layers can be class with static method
- Helper layer class can be a bean as per the use case
- Helper layer should have dependency on other helper layers and no other service dependency specifically

## Database Layers
- database layer is subject to database type
- funtions should be straight forward in service layer
- database layer with complex query should be defined using separate query util classes
- database layer should be responsible for fetching and executing query

## Application Property Configuration
- we should follow 'yml' syntax
- application properties - for common property across environments
- application-<env>.properties - for environment specific properties
## package structure
- package structure should follow feature packages if project is feature driven, common configs and utils should not be part of feature packages
- package structure for single responsibility usecase should use componenet packages like controller, service, etc

## build tools
- gradle project management tool is most preferred overall

## logging
- logging frameworks should be used for application logging with logback.xml and Sl4J 