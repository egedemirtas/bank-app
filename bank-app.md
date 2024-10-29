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

- build image `mvn compile jib:dockerBuild`
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
        2. Not scalable: when the number of instances increases, it becomes challenging to handle config for each
           instance
        3. Immutable: if you want to change a property, you need to restart the app

## Spring Cloud Config

- Provides externalization of config in distributed systems, solves all drawbacks of spring profiles
- There are two core elements:
    1. **Central repositories**: where properties are stored; can be github repo, database, file
    2. **Spring cloud server**: loads all the configs from central repository

## How to Build a Config Server (retrieves config from GitHub repo)

> [!IMPORTANT]
> To build config server, previous implementation in spring profiles has been changed
> 1) accounts -> `application.yml` & `application-prod.yml` & `application-qa.yml` has been moved to bank-config-server
> 2) The name of the files has been changed to `accounts.yml` & `accounts-prod.yml` & `accounts-qa.yml` respectively
> 3) Create same folders for loans and cards microservices in bank-app-config-server

1. A repo in GitHub will store our configs, then config server will retrieve these configs
    - created repo: `https://github.com/egedemirtas/bank-app-config`
    - Notice that this repo only has the environment-specific yml files
2. Create a maven project for your config server:
    - add `config server` as dependency from `spring.io` (adding `actuator` might also be beneficial for monitoring
      later)
    - add `@EnableConfigServer` to your main
    - inside `application.yml` add properties to retrieve data from config repo(created in step 1)
    - Test config server quickly with `http://localhost:8071/accounts/prod` (you should see the configs are retrieved
      from GitHub repo)
3. Create Dto classes inside each microservice (Ex: `AccountsContactInfoDto`) and create and endpoint inside controller
   classes to retrieve configs:
   `AccountController.getAccountInfo`, `LoanController.getLoanInfo`, `CardController.getCardInfo`
    - Remember to add `@EnableConfigurationProperties(value = {myBean.class})` to your main
4. For each ms, change `application.yml` to retrieve configs from config server:
    ```yaml
    spring:
      application:
        name: "accounts" # tell ms to look for config files starting with "accounts"
      profiles:
        active: qa
      config:
      # "configserver" is used no matter what your config server name is or its repo name
        import: "optional:configserver:http://localhost:8071"
    ```
5. Add `config client` dependency to each ms inside `pom.xml`

## How to Encrypt Configs?

1. Add an encryption key to your config-server yml:
    ```yml
    encrypt:
      key: "thisIsAStrongPassword"
    ```
2. Send the plain text as `POST` to your config server with body in raw text (not
   JSON !!): `http://localhost:8071/encrypt`
3. Config server will get your plain text, cipher it with the key in the yml file and return it as a response
4. You can now go to your config repo and paste cipher text in this format: **"{cipher}cipherText"**

- You can see the example in bank-app-config -> accounts-prod.yml

## How to Update Configs at Runtime?

- For all the classes that use configs for setting its fields:
    - These classes cannot be `record` or have immutable(`final static`) fields
    - This means that our previous implementation of `AccountsContactInfoDto`, `CardsContactInfoDto`,
      `LoansContactInfoDto` will change, the fields should be `private` with getters and setters

### Using Actuator

1. You need to have `actuator` dependency in your microservices
2. Add this to your microservices that use configs:
   ```yml
    management:
        endpoints:
            web:
                exposure:
                    include: "*" # this enables and exposes all the endpoints supported by actuator,
                    # include: "refresh"  # you can also specify only the refresh endpoint
    ```
3. Now that all actuator api is exposed, you can see all the endpoints.
   At this point when you change a config in **config repo**, changes will be reflected to
   **config-server**(`http://localhost:8071/accounts/prod`) but not to the main microservices
4. **Refresh** endpoint at the actuator should be accessed with POST with empty body.
   `http://localhost:8080/actuator/refresh` will refresh the configs for Accounts microservice.
   The response will include the changed configs and also `config.client.version` because the version is upgraded
   everytime a config is changed
5. After you call refresh, microservices can now get the latest config

> [!IMPORTANT]
> This approach requires to call `/actuator/refresh` endpoint everytime a config is updated. CI/CD can automate this
> with scripts or other approaches can be tried

### Using Spring Cloud Bus

What is ***Spring Cloud Bus***?

- Spring Cloud Bus links nodes of a distributed system with a lightweight message
  broker.
- This can then be used to broadcast state changes (e.g., configuration changes) or other management instructions.
- AMQP and Kafka broker implementations are included with the project

1. For this demo `RabbitMQ`'s Docker image is used rather than installing with Brew
    ```bash
   docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management
    ```
2. Spring Cloud Bus and RabbitMQ dependencies have to be added to all microservices (config server included):
    ```xml
   	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
    ```
3. You need to expose the bus refresh path in each microservice yml file (all the endpoints are already exposed in the
   previous section, see `Using Actuator`)
4. Provide RabbitMQ connection details for each microservice:
    ```yml
    spring:
      rabbitmq:
        host: "localhost"
        port: 5672
        username: "guest"
        password: "guest"
    ```
    - ***These are default values, RabbitMQ connection will still be achieved even if these default settings are not
      written***

> [!IMPORTANT]
> Bus refresh api path has to be invoked once for each microservice.
> All the instances of such a microservice will be able to retrieve the updated configs at runtime

### Using Spring Cloud Bus & Spring Cloud Config Monitor

1. The dependencies in previous implementation should exist (RabbitMQ & Spring cloud bus).
   Additionally, add this to your config-server:
    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-monitor</artifactId>
    </dependency>
    ```
2. This implementation does not use actuator paths.
   It uses monitor API paths (this has to be exposed just like actuator paths).
   Thus add this to the config-server `application.yml`
    ```yml
    spring:
      rabbitmq:
      host: "localhost"
      port: 5672
      username: "guest"
      password: "guest"
    
    management:
      endpoints:
        web:
          exposure:
            include: "*"
    ```
3. This implementation uses a webhook.
   Whenever a change occurs in config repo, the monitor API path will be invoked inside config-server.
   As soon as the monitor API path receives the webhook request, it is going to invoke refresh event with RabbitMQ and
   spring cloud bus; all the connected microservices will receive the latest configs.
   To create a webhook in your
   **config-repo**:
    1. Go to GitHub->config-repo. Click Settings
    2. Webhook-> create new webhook
    3. This new webhook will publish an event whenever a change is pushed in this repo.
   ```txt
   Payload URL: http://localhost:8071/monitor (where the webhook request should be sent)
   Content type: JSON
   ```
   > [!IMPORTANT]
   > Notice that in our example payload URL is configured as "localhost"
   > It is impossible for GitHub to target our local system.
   > `https://console.hookdeck.com/` provides a solution for this.
   > ```bash
   >  brew install hookdeck/hookdeck/hookdeck
   >  hookdeck listen 8071 source --cli-path /monitor  (use config-server port and motinor API path)
   >  ```
   > This command will provide us the Payload URL that we can use instead of
   localhost: `https://hkdk.events/lu1xs6cuwfdhes`

### Updating Docker Compose File to Adapt Config Server Changes

- The main strategy is to use different docker-compose files for different environments.
  We already had a docker-compose file in `bank-app` and we will make adjustments on this file for default, QA and PROD.

1. Add config-server to docker-compose
    1. This is very similar to every other service declaration we did such as accounts, loans and cards in
       docker-compose
    2. We used these statements in our services' `application.yml`:
        ```yml
        config:
          import: "optional:configserver:http://localhost:8071"
        ```
        - We need to change `localhost` because our services will try to connect to config-server within it's own
          network
        - For each service that will use config-server, we need to this update in docker-compose:
          ```yml
          accounts: # do this for loans and cards
            environment:
              #configserver:http://{{config-server-docker-name}}:{{port-of-config-server-inside-docker-network}}
              SPRING_CONFIG_IMPORT: "configserver:http://config-server:8071" 
          ```
2. Add the profile you want to use for config-server. Ex:
    ```yml
    accounts: # do this for loans and cards
      environment:
        SPRING_CONFIG_IMPORT: "configserver:http://config-server:8071" 
        SPRING_PROFILES_ACTIVE: default # can be prod or QA as well
    ```
3. Force config-server to be started before other microservices

   > [!IMPORTANT]
   > ***Liveness and Readiness Probes***
   > container orchestration products like `Kubernetes` will handle elasticity and scaling.
   > To handle containers effectively, Kubernetes need to know whether my container is running without any issues.
   > Liveness probe: used to signal that container is alive or dead, Kubernetes will take action if container is dead
   > Readiness probe: used to signal that container is ready to start network traffic

   > [!IMPORTANT]
   > In spring boot, actuator gathers liveness and readiness info from `ApplicationAvailability`
   > This info is used in dedicated health indicators: LivenessStateHealthIndicator & ReadinessStateHealthIndicator
   > These indicators are shown in the global health endpoint: `/actuator/health/liveness`

    1. For these reasons you need to have `spring-boot-starter-actuator` dependency in config-server and you need to
       expose health info:
       ```yml
       management:
         readiness-state:
           enabled: true
         liveness-state:
           enabled: true
       endpoint:
         health:
           probes:
             enabled: true
       ```
        - Test with `http://localhost:8071/actuator/health`, `http://localhost:8071/actuator/health/liveness`,
          `http://localhost:8071/actuator/health/readiness`
    2. Now we need to mention these in our docker-compose, so that Kubernetes can understand:
         ```yml
         config-server:
           healthcheck:
             test: "curl --fail --silent localhost:8071/actuator/health/readiness | grep UP || exit 1"
             interval: 10s # if command failed: retry after 10s
             timeout: 5s # wait for response up-to 5s
             retries: 10 # if command failed: retry up-to 10 times
             start_period: 10s # run the health check command after 10s
         ```
    3. Add config-server as a dependency to accounts, loans and cards in docker-compose:
       ```yml 
       accounts:
         depends_on:
           config-server:
             condition: service_healthy
       ```
    4. Since we use spring cloud bus, all of our services depend on rabbitMQ, we need to add rabbitMQ in docker-compose:
       ```yml 
       rabbit:
         image: rabbitmq:3.12-management
         host-name: rabbitmq # specific to rabbitMQ
         ports:
           - "5672:5672" # for activities
           - "15672:15672" # for management
         healthcheck:
           test: rabbitmq-diagnostics check_port_connectivity
           interval: 10s
           timeout: 5s
           retries: 10
           start_period: 5s
         networks:
           - bank-network
       ```
    5. Add rabbitMQ as a dependency to **only** config-server: (we need rabbitMQ for accounts/loans/cards as well
       but since compose flow will be rabbit->config->other services we don't need to explicitly define rabbit
       dependency in other services)
        ```yml
        depends_on:
          rabbit:
            condition: service_healthy
        ```
    6. Inside all our microservices, in `application.yml` we defined RabbitMQ connection details. The `host` part
       must be overridden so that our microservices won't target localhost. The other parts such as `port`,
       `username`, `password` ***may not be overridden in our case*** since these are default values for RabbitMQ
       connection.
        ```yml
        environment:
          SPRING_RABBITMQ_HOST: rabbit
        ```

___ 

# Optimizing Docker

- Some fields such as `network` are repeated in docker-compose, we need to optimize this by declaring them from one
  place.
  This will also enable us to make a change to a field in docker compose and affect all services that use it
- Created files and latest form of docker-compose can be found in `bank-app.docker-compose.prod`. Simply put common
  services are created in `common-config` and these are used with `extends` in docker-compose