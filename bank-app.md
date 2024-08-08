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
2. Send the plain text as `POST` to your config server with body in raw text (not JSON !!): `http://localhost:8071/encrypt`
3. Config server will get your plain text, cipher it with the key in the yml file and return it as a response
4. You can now go to your config repo and paste cipher text in this format: **"{cipher}cipherText"**
- You can see the example in bank-app-config -> accounts-prod.yml