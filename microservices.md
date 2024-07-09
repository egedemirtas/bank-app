# Microservices

- Imagine a bank app; it provides products such as Accounts, Cards, Loans

![mono-soa-micro.png](./mono-soa-micro.png)

## Monolith

- All the business functionality of a bank app is deployed as a single unit in a single server
- ***Presentation layer(HTML, JS, CSS)***, ***business logic layer(backend)***, ***data access layer*** and
  ***database*** are a single unit i a single server
- Monolith apps have only one database

### Advantages of Monolith

1. Simpler development and deployment for small apps and small teams
2. Fewer cross-cutting concerns since all your code deployed inside a single server:
    - non-functional requirements: security, auditing, logging
3. Might have better performance due to no network latency: a call from one service to another will be in the same
   server. There is no network call

### Disadvantages of Monolith

1. Difficult to adopt new tech: if you want to enhance your existing app with new, you might need to update your
   entire code base. Accounts team might want to use the latest framework, but there might be a pushback from
   the Cards/Loans
   team
2. Limited agility: you can't just decide to use a new framework in a day. Because your entire code base is tightly
   coupled
3. Single code base and difficult to maintain: since all of your code is a single unit, it will be hard to maintain as
   your apps grow with new features
4. Not fault tolerance: issues with scalability and availability
5. A tiny development will require a full deployment: your app will have downtime
6. No parallel development
7. Limited scalability: if you want to scale your app, you need to bring up one more server (even though only Accounts
   microservice needs scaling).

## Service Oriented Architecture (SOA)

- SOA emerged as an approach to combat challenges of large monolithic apps
- SOA segregates tightly coupled UI and backend logic:
    - server1 has UI and ESB; server2 has backend services
- For UI to communicate with backend services, ESB is needed
    - Whenever a request comes from the UI to ESB; ESB redirects it to the corresponding service
- Just like in monolith: services are tightly coupled to a single shared DB
- Similar to monolith: a tiny change in BE requires full deployment of BE (Realize that UI and BE are managed in
  different repos, thus a tiny change in BE doesn't require deployment of UI. Only BE needs full deployment)

### Advantages of SOA

1. Reusability of services
2. Better maintainability
3. Higher reliability
4. Parallel development

- In SOA, all your logic is separated into components;
    1. all accounts related logic will be inside the account service.
    2. UI code is separated from backend code. They will live in different servers as well.

    - This is the root reason for all these advantages

### Disadvantages of SOA

1. Complex management: have to manage the communication between services
2. High investment cost to set up an ESB
3. Extra overload: ESB is a component that has to be maintained just for UI-BE communication
4. Parallel development between UI-BE (but there is no parallel development between services in BE)
5. Limited scalability: if you want to scale your app, you need to bring up one more server for BE services and also one
   for ESB

## Microservices

- Microservices are independently deployable services that are built around business domain
- A service encapsulates functionality and makes it accessible to other services via network
- Each microservice has its own DB
- Each microservice will be deployed into their own servers/containers

### Advantages of Microservices

1. Easy to develop, test, deploy: because you loosely coupled all your business logic into separate small components
2. Increased agility: since there is no tight coupling between services, dev teams will have their own
   enhancement/deployment lifecycle
3. Ability to scale horizontally: you can scale a single microservice with containerization
4. Parallel development
5. Modeled around business domain
6. Each microservice can use a different framework, language, DB

### Disadvantages of Microservices

1. Complexity: your services will be in many different containers and clusters. It will be hard to manage communication
   between these services
2. Infrastructure overhead: you are going to deploy your services to multiple servers and containers, however, you still
   need to monitor them which creates infrastructure overhead
3. Security concerns: UI to service, service to service communication will be done with web services which increase
   security concerns. In monolith, communication was done with simple method calls

## Why Use Spring Boot for Microservices?

1. You can build executable JAR files instead of traditional WAR/EAR files. The JAR file contains all the dependencies
   and configs required to run the microservice
    - Apps that are packaged as WARs rely on the presence of a server installed in the execution environment for
      running. That server also needs to have JVM installed.
    - However, a self-contained JAR (fat-JAR/uber-JAR) already contains an embedded server in it.
2. Provides built-in features and integrations: auto-config, dependency injection
3. Provides embedded Tomcat/Jetty server, which can run the microservice directly, without the need for separate server
   installation
4. Built-in production-ready features: metrics, health check, externalized config
5. Enables to quickly bootstrap a microservice project and start coding. There are Starter dependencies that provide
   preconfigured settings for databases, queues, etc. (Before spring boot, you need to do a lot of config&dependency
   management to set up a DB for a service)
6. Suited for cloud native development. Integrates well with Kubernetes, Docker. You can also deploy to them cloud
   providers easily

___

# Building Bank App

## Handling Exceptions

1. Create a class for each custom exception:
    - Notice that it is extending from `RuntimeException`
    - notice that you can use any HTTP Status
    ```java
    @ResponseStatus(value = HttpStatus.BAD_REQUEST)
    public class CustomerAlreadyExistsException extends RuntimeException {

        public CustomerAlreadyExistsException(String message) {
            super(message);
        }
    }
    ```

2. Create a handler class that will manage **all** the exceptions in your app:
   > In the course `@ControllerAdvice` is used however `@RestControllerAdvice` is a better option for simplicity
   > `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. We can use in REST web services.
   > `@ControllerAdvice`: We can use in both MVC and Rest web services, need to provide the ResponseBody if we use this
   in Rest web services.
    - Notice that you must annotate your handler class with `@ControllerAdvice`
    - Notice that you have to specify the exception class that you want to
      handle: ` @ExceptionHandler(CustomerAlreadyExistsException.class)`
    - When an exception is thrown, our handler class will look for the most specific class related to exception
    - Notice that `ErrorResponseDto` is a custom class created for error related responses
    ```java
    @ControllerAdvice
    public class AccountExceptionHandler extends ResponseEntityExceptionHandler {

        @ExceptionHandler(CustomerAlreadyExistsException.class)
        public final ResponseEntity<ErrorResponseDto> handleDuplicateCustomerException(Exception ex,
                                                                                    WebRequest webRequest) {
            ErrorResponseDto errorResponseDto =
                    new ErrorResponseDto(webRequest.getDescription(false), HttpStatus.BAD_REQUEST,
                            ex.getMessage(), LocalDateTime.now());

            return new ResponseEntity<>(errorResponseDto, HttpStatus.BAD_REQUEST);
        }
    }
    ```

## Auditing

1. Create date, created by etc. should be in `BaseEntity`:
    - `@MappedSuperclass`: other entities will use `BaseEntity` as parent. We want to inherit audit fields, and also we
      don't want to create a separate table for `BaseEntity`
    - `@EntityListeners(AuditingEntityListener.class)`: mandatory for audit
    - `@CreatedDate`: will audit create date
    - `@Column(updatable = false)`: prevents update of `createdAt` when updating a record, you must use this
      with `@CreatedDate`
    - `@LastModifiedDate`: will audit update date
    - `@Column(insertable = false)`: prevents insertion of `updatedAt` when creating a record, you must use this
      with `@LastModifiedDate`
    ```java
    @MappedSuperclass
    @Getter
    @Setter
    @EntityListeners(AuditingEntityListener.class)
    public class BaseEntity {

        @CreatedDate
        @Column(updatable = false) //don't update this when a record is updated
        private LocalDateTime createdAt;

        @LastModifiedDate
        @Column(insertable = false) // don't provide these columns when INSERT
        private LocalDateTime updatedAt;

        // same logics for updateBy and createBy
   }
    ```
2. Create a class for auditing current user:
    ```java
    @Component("auditAwareImpl") // Can be any name. You have to use this name in 'AccountsApplication'
    public class AuditAwareImpl implements AuditorAware<String> {

        @Override
        public Optional<String> getCurrentAuditor() {
            return Optional.of("ACCOUNTS_MS");
        }
    }
    ```
3. Enable jpa audit in your main: `@EnableJpaAuditing(auditorAwareRef = "auditAwareImpl")`
    - Notice that `"auditAwareImpl"` comes from the name we gave to `AuditAwareImpl` bean

## Documentation of REST APIs with `springdoc-openapi`

- `https://springdoc.org/`
- Go to `http://localhost:8080/swagger-ui/index.html`
- Enhancing documentation:
    1. Add `@OpenAPIDefinition` to your main to provide general info about your app
        - You can enhance `@OpenAPIDefinition` with many options: `@Info(description, version, contact, license)`,
          `@ExternalDocs`
    2. Add `@Tag` to your controller to provide info about your controller (`AccountsController`)
    3. Add `@Operation` to your methods in your controller class to provide info about your methods
       (`AccountsController.createCustomerAndAccount`)
    4. By default, all your methods in your controller is shown to return `HTTP 201` in swagger, you can change this by
       adding `@ApiResponse` to your methods (`AccountsController.createCustomerAndAccount`)
        - What if a method may return 200, 201, 204? You can use `@ApiResponses` and add many `@ApiResponse` as you
          would like
        - As of now realize that `ErrorResponseDto` is missing in swagger, this is because this DTO is not returned
          by any method in controller. This DTO is used when exception is thrown. There is a way to include this
          class in swagger: use `@Content` with `@ApiResponse`
    5. By default, your Schemas in swagger are seen as `AccountDto` (schema names are from class names). You can
       change this by adding `@Schema` in your DTOs at class level (`CustomerDto`)
        - You can go one step further and add `@Schema` at field level in your DTOs

___

# Docker Containerization

## Build Image of Accounts with Dockerfile

- `FROM`:
    - Used to tell to Docker server that my image needs another image as a dependency
    - The instructed image will be imported to our image whenever we want to generate our image
- Our microservice needs Java as a dependency: `FROM openjdk:17-jdk-slim` (`FROM image:tag`)
- `MAINTAINER ege.com`: specify the maintainer of the image we are building
- `COPY target/accounts-0.0.1-SNAPSHOT.jar accounts-0.0.1-SNAPSHOT.jar`: copy the jar in target(our local) and copy
  it to the new image
- `ENTRYPOINT ["java", "-jar", "accounts-0.0.1-SNAPSHOT.jar"]`: whenever a container is ordered to be generated from
  this image execute this command. You can actually run your jar in your local by executing `java -jar accounts-0.0.
  1-SNAPSHOT.jar` in your terminal

```dockerfile
FROM openjdk:17-jdk-slim
MAINTAINER https://github.com/egedemirtas
COPY target/accounts-0.0.1-SNAPSHOT.jar accounts-0.0.1-SNAPSHOT.jar
ENTRYPOINT ["java", "-jar", "accounts-0.0.1-SNAPSHOT.jar"]
```

- Add this to pom:

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

- Build image with `docker build . -t rarewind/bank-accounts:v1`
    - The convention is `-t usernameAtDockerHub/yourAppName:versionOrTagName`
- Run image with `docker run -d -p 8080:8080 rarewind/bank-accounts:v1`
    - `-p port1:port2`:
        - `port1`: expose my app to the outside world at 8080
        - `port2`: my docker container will run at 8080 (remember that Account app is running at 8080)

## Build Image of Loans with Buildpacks

- you need to adjust your pom:

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>rarewind/bank-${project.artifactId}:v1</name>
        </image>
    </configuration>
</plugin>
```

- Execute `mvn spring-boot:build-image` on maven plugin
- Other images related to Buildpacks will be generated
- Run image: `docker run -d -p 8090:8090 rarewind/bank-loans:v1`

## Build Image of Cards with Google Jib

- Go to `https://github.com/GoogleContainerTools/jib`, select
  maven: `https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin`
- add plugin:

```xml

<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.3</version>
    <configuration>
        <to>
            <image>rarewind/bank-${project.artifactId}:v1</image>
        </to>
    </configuration>
</plugin>
```

- build image `mvn compile jib:dcokerBuild`
- Run image `docker run -d -p 9000:9000 rarewind/bank-cards:v1`

## Push Images to DockerHub

```bash
docker image push docker.io/rarewind/bank-accounts:v1
docker image push docker.io/rarewind/bank-loans:v1
docker image push docker.io/rarewind/bank-cards:v1
```

## Run Images With Docker Compose

- `docker compose up -d`

```yml
services:
  accounts:
    image: rarewind/bank-accounts:v1
    container_name: bank-accounts-ms
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 700m
    networks:
      - bank-network
  cards:
    image: rarewind/bank-cards:v1
    container_name: bank-cards-ms
    ports:
      - "9000:9000"
    deploy:
      resources:
        limits:
          memory: 700m
    networks:
      - bank-network
  loans:
    image: rarewind/bank-loans:v1
    container_name: bank-loans-ms
    ports:
      - "8090:8090"
    deploy:
      resources:
        limits:
          memory: 700m
    networks:
      - bank-network
networks:
  bank-network:
    driver: "bridge"
```

--- 

# Config Server

## Why do we need config-server?

- If you store your config in your business logic, you need to build different images for different environments
- For more details, you can check the 12-factor app

## How does configuration work in Spring Boot?

- By default, Spring Boot looks for application.properties or application.yml file in the classpath
- Order of priority in decreasing order: command line argument > java system properties > OS environment variables >
  application.properties

## How to read properties?

1. `@Value`:
    - You can use it to directly inject into your beans:
   ```java
   @Value("${property.name}")
   private String value;
   ```
2. `Environment`:
    - You can only access to environment variables but not application.properties
    - This interface provides methods to access properties:
   ```java
   @Autowired
   private Environment env;
   
   public void getProperty(){
        String propertyVal = env.getProperty("JAVA_HOME"); //or MAVEN_HOME
   }
   ```
3. `@ConfigurationProperties`
    - Recommended approach since it avoids property names
    - Enables binding of entire groups of properties to a bean
   ```java
   @ConfigurationProperties(prefix = "myPrefix")
   public class MyConfig{
        private String property;
   }
   ```
    - You also need to add `@EnableConfigurationProperties(value = {myBean.class})` to your main
    - Example: accounts->`AccountController`->`getAccountInfo()`
    - More usage styles: https://www.baeldung.com/configuration-properties-in-spring-boot

## Spring Profiles

- How to read properties based on the environment?
- The default profile is always active (properties in `application.yml` will be used), we can create profiles by
  creating files such as
  `application-prod.properties`, then we can activate a profile with property: `spring.profiles.active=prod`
- Example: accounts->`application-qa.yml`, `application-prod.yml` and also the changes in `application.yml`
- **Major Disadvantage**: you need to make different builds for each environment, this is against 15 factor-app
    - You can solve this with CLI arguments, JVM properties, environment variables, but using these will have their own
      drawbacks:
        1. Error-prone: These involve executing commands and manually setting up the application
        2. Not scalable: when the number of instances increases, it becomes challenging to handle config for each instance
        3. Immutable: if you want to change a property, you need to restart the app

## Spring Cloud Config

- Provides externalization of config in distributed systems, solves all drawbacks of spring profiles
- There are two core elements:
    1. **Central repositories**: where properties are stored; can be github repo, database, file
    2. **Spring cloud server**: loads all the configs from central repository

## How to Build a Config Server
- add `config server` as dependency from spring.io (adding `actuator` might also be beneficial for monitoring later)
- add `@EnableConfigServer` to your main