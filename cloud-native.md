# Cloud Native Apps & 15 Factor Methodology

## What is Cloud Native App

- Cloud native apps are designed specifically to be run on cloud. They leverage the advantages of cloud computing tech
  and services:
    - scalability
    - elasticity
    - flexibility
- Containers, microservices, declarative APIs exemplify this approach
- These techniques enables loosely coupled systems that are ***resilient***, ***manageable*** and ***observable***
- Combined with ***robust automation***, engineers can make high impact changes frequently and reliably with minimal
  toil

## Characteristics of Cloud Native Apps

1. **Microservices**: app is broken down to ***smaller loosely coupled services*** that can be developed, deployed,
   scaled independently
2. **Containers**: provide light-weight environments to run apps. Which makes them highly portable across different
   cloud providers
3. **Scalability & Elasticity**:
    - ***designed to scale horizontally***: allows apps to handle an increased load by adding more instances of services
    - ***Can be automatically scaled down or up*** based on demand with cloud-native orchestration platforms like
      Kubernetes
4. ***DevOps Practices***:
    - promotes collaboration between development and devops teams
    - often promote ***CI/CD***:
        - ***CI/CD accelerates the software development lifecycle***
        - ***CI***: refers to the practice of automatically and frequently integrating code changes into a shared source
          code
          repository
        - ***CD*** is a 2-part process that refers to the integration, testing, and delivery of code changes
5. ***Resilience / Fault Tolerance***: utilize techniques like distributed architecture, load balancing and
   automated fail recovery to ensure high availability and fault tolerance
6. ***Cloud Native Services***: take advantage of cloud-native services provided by the cloud platform: messaging
   queues, caching system and identity services. This allows devs to focus more on app logic

## Cloud Native Apps VS Traditional Enterprise Apps

1. ***Predictable behavior***: you can track an issue in microservice environment because all your business
   logic is
   loosely coupled and separated into multiple microservices. However, in monolithic apps since all of your logic is
   clubbed together, a developer needs to debug lots of code to understand where the problem is.
2. ***OS abstraction***: containerization allows cloud native apps to be run on any cloud provider, regardless of
   their OS.
3. ***Right-sized capacity and independent***: cloud native apps follow microservices which are independent services
   with correct boundaries
4. ***CI/CD***: cloud native apps support CI/CD, agile. Traditional apps mostly support the waterfall model
5. ***Rapid recovery & automated scalability***: Kubernetes supports recovery of failed services and also depending
   on the load supports instance creation of services. However, traditional apps don't use Kubernetes, thus there won't
   be any automated recovery and scalability

## 15 Factors

- These are development principles of cloud native apps. If you apply these principles, you will achieve:
    1. Cloud platform deployment: apps designed to be seemly deployed on various cloud platform
    2. Scalability as core attribute
    3. System portability
    4. Enable continues deployment and agility

1. **One codebase one app**:
    - ***What is it?***
        - Codebase: any set of repos who share a root commit
        - One codebase is tracked in version control.
        - If we have multiple codebases, it’s not an app but a distributed system; each will comply with 12-factor
          separately
        - If multiple apps share the same code, it violates the fifteen-factor.
        - There is only one codebase per app, but there are many deployments of the app.
        - The codebase is the same across all deployments, although different versions may be active in each deployment.
    - ***Why important?***
        - A unified codebase fosters consistency, making it easier for developers to work together seamlessly.
        - It ensures that everyone is on the same page, using the same version of the code.

2. **Dependency Management**:
    - ***What is it?***
        - The goal of managing dependencies is to ensure consistent and reproducible builds across different
          environments.
        - A fifteen-factor app declares all dependencies, completely and exactly, via a dependency declaration manifest.
        - It simplifies the setup for developers new to the app.
        - The new developers will set everything up by running the app’s code with a deterministic build command.
        - A fifteen-factor app doesn't rely on the implicit existence of any system tools.
        - If the app needs to shell out to a system tool, that tool should be vendored into the app.
    - ***Why important?***
        - By managing dependencies effectively, you create a predictable and reproducible environment.
        - Without dependency management, different developers may use different dependencies or versions which will
          lead to bugs, compatibility issues
    - ***How to implement?***
        - From Java perspective, Maven and Gradle are the examples of dependency manager, which allows users to specify
          the dependencies and also to exclude the implicit dependencies which should not be the part of the system.

3. **Config**:
    - ***What is it?***
        - Never store constants in the code.
        - To be fifteen-factor compliant requires a strict separation of config from code.
        - Config varies across deployments; code doesn't.
        - Always prefer storing the config in environment variables or externalize the configuration from the
          application
    - ***Why important?***
        - Separating configuration from code enhances flexibility.
        - It allows you to change settings without modifying the codebase.
        - This is particularly crucial when deploying an application in different environments, such as development,
          testing, and production.
    - ***How to implement?***
        - In the Java world, one of the examples of externalizing the application configuration is the use of
          Spring-Cloud-Config.

4. **Backing Services**:
    - ***What is it?***
        - **Backing Service**: any service the app consumes over the network as part of its normal operation.
        - The principle states that all the backing services, either local or third-party services (the principle
          indicates that there is no distinction between local or third party services), should be treated as
          **attached resources**
        - The attached resources can be swapped at any point in time without impacting the service.
        - This means your application should be designed to connect to these services as if they were local components,
          regardless of whether they run in the same environment or are provided as third-party services.
    - ***Why important?***
        - By treating data storage, messaging, and caching as attached resources, you decouple your application from the
          specifics of the service implementation.
        - This enhances flexibility, scalability, and maintainability.
    - ***How to implement?***
        - Use configuration and environment variables to abstract the details of backing services.
        - The idea is to make it easy to switch between different service providers or configurations without modifying
          your code.

5. **Build, Release, Run**:
    - ***What is it?***
        - The build stage is a process that converts a code repo into an executable bundle known as a build.
        - The release stage takes the build and combines it with the deployment’s current config.
        - The run stage runs the app in the execution environment.
        - Each stage is strongly separated. For instance, any code changes are not possible at runtime as there is no
          way to propagate those changes back to
          the build stage.
    - ***Why important?***
        - It becomes easier to track changes, maintain consistency across environments, and automate deployment
          processes.
        - Each stage has its responsibilities, enabling a more organized and efficient development pipeline.
    - ***How to implement?***
        - Use a build tool to compile your code and package it into a deployable artifact: Maven

6. **Processes**:
    - ***What is it?***
        - Fifteen-factor processes are ***stateless*** and ***share-nothing***. We have to use a stateful backing
          service whether we want to store any data.
        - Never use sticky sessions. Never assume that anything cached in memory or disk will be available on a future
          request or job.
        - In other words, the application should not rely on the persistence of data or state between successive
          requests. This design approach simplifies scaling and enhances reliability.
    - ***Why important?***
        - Stateless processes are crucial for scalability.
        - When your application doesn't depend on the state between requests, you can easily scale horizontally by
          adding more instances of your application.
        - Stateless processes are also more resilient, as they can recover quickly from failures.
    - ***How to implement?***
        - For instance, by making applications aligned with the stateless behavior of the REST, the services can be
          horizontally scaled as per the requirements without any impact.
        - Requests will be stateless and not rely on shared in-memory state. State will be stored externally in a
          database

7. **Port Binding**:
    - ***What is it?***
        - Export services via port binding
        - This means your application should be self-contained, explicitly declaring the ports it uses to provide
          services. Thus, your app will not rely on the runtime injection of a webserver into the
          execution environment to create a web-facing service.
        - This approach contributes to the simplicity, flexibility, and ease of deployment.
    - ***Why important?***
        - By declaring the port your application uses, you make it clear to both the development environment and the
          runtime where to access your services.
        - This is crucial for the portability of your application across different platforms and environments.
    - ***How to implement?***
        - From a Java perspective, Spring boot is one of the examples as it by default comes with an embedded server.

8. **Concurrency**:
    - ***What is it?***
        - The Concurrency factor emphasizes the idea of scaling your application by adding more processes.
        - Rather than relying on a single, monolithic instance, breaking your application into smaller, concurrent
          processes allows for better resource utilization, improved responsiveness, and enhanced scalability.
    - ***Why important?***
        - Concurrency enables your application to handle increased demand by distributing the workload across multiple
          processes.
        - This approach also contributes to fault tolerance and responsiveness, as individual processes can
          handle requests independently.
    - ***How to implement?***
        - Design your application to operate as a collection of independent, stateless processes.
        - This can be achieved by Docker and Kubernetes
9. **Disposability**:
    - ***What is it?***
        - The Disposability factor encourages the design of applications that can start and stop quickly.
        - Treating processes as disposable entities means that individual components of your application can be easily
          and rapidly replaced, contributing to fault tolerance and responsiveness.
        - Processes should also be robust against sudden death.
        - Graceful shutdowns are crucial and the system must
          ensure the correct state.
    - ***Why important?***
        - In cloud-native environments, the ability to quickly start and stop processes is crucial for maintaining high
          availability and responding to dynamic workloads.
        - Disposable processes contribute to easier scaling, resilience to failures, and efficient resource utilization.
    - ***How to implement?***
        - Use containerization and Kubernetes

10. **Dev/prod parity**:
    - ***What is it?***
        - The fifteen-factor app is designed for continuous deployment by keeping the gap between development and
          production small.
        - The time between deployments should be just hours than weeks.
        - The author of the code and who deploys the application is the same person.
        - Dev env and prod env should be as similar as possible.
        - The principle also resists using different backing services between development and production (don't use
          Kafka in Dev and RabbitMQ in Prod)
    - ***Why important?***
        - Discrepancies between development and production environments can lead to a variety of issues, such as
          differences in behavior, unanticipated bugs, and deployment challenges.
    - ***How to implement?***
        - You can use Docker to package your app and its dependencies

11. **Logs**:
    - ***What is it?***
        - The Logs factor advocates for treating logs generated by your application as event streams.
        - Rather than writing logs to a file or a database, stream them in real-time to a central logging system
        - The principle suggests separating the two processes — log generation and processing the log’s information.
    - ***Why important?***
        - Logs are invaluable for understanding the behavior of your application, identifying issues, and
          troubleshooting problems.
        - By treating logs as event streams, you gain the advantage of real-time insights, making it easier to detect
          and respond to issues promptly.
    - ***How to implement?***
        - Integrate logging into your application's code, stream these logs to a centralized logging service for
          indexing and analysis like Splunk to find past events, graph trends, and active
          alerting.

12. **Admin Processes**:
    - ***What is it?***
        - The Admin Processes factor advocates for treating administrative or management tasks as separate, one-off
          processes rather than bundling them with the main application.
        - This approach ensures better isolation, security, and maintainability of tasks related to database migrations,
          data imports, or other administrative activities.
    - ***Why important?***
        - Admin processes often involve tasks that are not part of the regular application runtime but are crucial for
          maintaining the system.
        - By running these tasks as separate processes, you reduce the risk of interfering with the main application's
          performance, enhance security, and ensure a clean separation of concerns.
    - ***How to implement?***
        - Identify tasks that are administrative in nature, such as database migrations, data imports, or configuration
          updates.
        - Create standalone scripts or executables for these tasks, ensuring they can be executed independently of the
          main application.
        - Trigger these admin processes as needed

13. **API First**:
    - ***What is it?***
        - API First principle implies defining the service contract first to help consumers understand what the
          request and response communication is expected to be.
        - The service consumers can work in parallel to develop the consuming applications.
        - All this can happen even
          before the actual service contract is implemented and is made available.
    - ***Why important?***
        - Teams Can Develop in Parallel and Know What to Expect
        - This approach enables consumers and service developers to work in parallel.
        - It helps avoid bottlenecks and facilitates the virtualization of the APIs so the consumers can run the tests
          against the mocks.
    - ***How to implement?***
        - You can use software that provides mock servers for your API

14. **Telemetry**:
    - ***What is it?***
        - The principle focuses on design to include the collection of monitoring domain/application-specific logs/data,
          health information, or more statistics on modern cloud platforms.
    - ***Why important?***
        - Monitoring our microservices, containers, and env help us to scale, self-heal and manage alerts for end-users
          and platform operators.
        - We can use machine learning towards those metrics to derive future business strategies.
        - We can observe the application’s performance, stream of events and data, health checks, when the application
          starts, scales, and so on.
    - ***How to implement?***
        - Grafana, Prometheus

15. **Authentication & Authorization**:
    - Make sure all security policies are in place. Developers tend to underestimate the importance that security has.
    - APIs should be secured using OAuth, RBAC, etc.
    - Web content should be exposed externally on HTTPS
    - MFA
    - Database protection