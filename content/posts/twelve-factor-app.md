---
title: "Twelve Factor App"
date: 2022-03-06T08:06:08Z
---

The [Twelve-Factor methodology](https://12factor.net/) is a set of twelve best practices to develop applications developed to run as a service.  
This was originally drafted by Heroku for applications deployed as services on their cloud platform, back in 2011.  
Over time, this has proved to be generic enough for any software-as-a-service (SaaS) development.  

Docker ecosystem is awesome for developing twelve-factor apps.  

Container images offer a contract with the operating system that can be effectively identical between your development environment and production environment.  

This guide will show you how to develeop a [Spring Boot](https://spring.io/projects/spring-boot) application based on this methodology.  

## II. Dependencies — Explicitly declare and isolate dependencies

Docker images are built from explicit Dockerfile recipes and Docker containers are run as isolated environments.

Dockerfile offers a way to explicitly declare the base operating system (FROM), and to run commands to install additional system packages and application dependencies (RUN).

With these tools you can say that you want an Alpine 3.15 OS, Java 11 etc.  

## III. Config — Store config in the environment

Containers rely heavily on linux environments for configuration.  

During development you can use [Compose Spec](https://www.compose-spec.io/) to explicitly define what environment variables will be set in a container.  

Additionally there is the Dockerfile ENV instruction and the docker run –env=[] and docker run — env-file=[] runtime options.

## VII. Port binding — Export services via port binding

Containers rely heavily on port binding.  

compose.yml has a [ports array](https://github.com/compose-spec/compose-spec/blob/master/spec.md#ports) that explicitly defines a HOST:CONTAINER port mapping.  
docker run -p HOST:CONTAINER lets you define this at runtime.  

```docker
services:
  whoami:
    image: traefik/whoami
    ports:
      - target: 8080
        host_ip: 127.0.0.1
        published: 8080
        protocol: tcp
```

## IV. Backing Services — Treat backing services as attached resources

Docker containers share little to nothing with other containers and therefore should communicate with backing services over the network.  

With these tools you can say that your application depends on a backing Postgres 9.4 and Redis 3.0 service and have your application connect to them via hostname and port information present in their environment.  

## VI. Processes — Execute the app as one or more stateless processes

By default, containers are shared nothing processes with ephemeral storage.  

docker-compose.yml defines a collection of services, each one with its own image or build recipe and command.  

With this tool you can say that your application has a both a web and worker process.  

<!--adsense-->

## Let's do it

Let's now define a simple application that we'll try to develop with the tools and practices we just discussed.  
We all love watching movies, but it's challenging to keep track of the movies we've already watched.  

![image alt text](/images/12-factor/movies-app.jpg)

This is quite a simple and standard microservice with a data store and REST endpoints.  
We need to define a model which will map to persistence as well:

```java
@Entity
public class Movie {
    @Id
    private Long id;
    private String title;
    private String year;
    private String rating;
    // getters and setters
}
```

We've defined a JPA entity with an id and a few other attributes.  
Let's now see what the REST controller looks like:  

```java
@RestController
public class MovieController {
 
    @Autowired
    private MovieRepository movieRepository;
    @GetMapping("/movies")
    public List<Movie> retrieveAllStudents() {
        return movieRepository.findAll();
    }

    @GetMapping("/movies/{id}")
    public Movie retrieveStudent(@PathVariable Long id) {
        return movieRepository.findById(id).get();
    }

    @PostMapping("/movies")
    public Long createStudent(@RequestBody Movie movie) {
        return movieRepository.save(movie).getId();
    }
}
```

Next, the twelve-factor app should always explicitly declare all its dependencies.  
We should do this using a dependency declaration manifest, using maven we have to add:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

A twelve-factor app should externalize all such configurations that vary between deployments.  
The recommendation here is to use environment variables for such configurations.    
This leads to a clean separation of config and code.

```properties
spring.datasource.url=jdbc:mysql://${MYSQL_HOST}:${MYSQL_PORT}/movies
spring.datasource.username=${MYSQL_USER}
spring.datasource.password=${MYSQL_PASSWORD}
```

Create a Dockerfile like this one:


```docker
FROM openjdk:8-jdk-alpine as build
WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw install -DskipTests
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)

FROM openjdk:8-jdk-alpine

# Default env variables value
ENV MYSQL_HOST=mysql \
    MYSQL_USER=admin \
    MYSQL_PASSWORD=s3cret # In production we have to find a different solution

VOLUME /tmp
ARG DEPENDENCY=/workspace/app/target/dependency
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","it.alvise.movies.Application"] # Java Main
```

We are redy to declare our application using a compose file:

```docker
# Names our volume
volumes:
  my-db:

services:
  movie-app:
    build:
      context: .
      dockerfile: ./Dockerfile
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_USER: admin
      MYSQL_PASSWORD: s3cret
    ports:
      - target: 8080
        host_ip: 127.0.0.1
        published: 8080
        protocol: tcp
    depends_on:
      - db

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: 'movies'
      # So you don't have to use root, but you can if you like
      MYSQL_USER: 'admin'
      # You can use whatever password you like
      MYSQL_PASSWORD: 's3cret'
      # Password for root access
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      # <Port exposed> : < MySQL Port running inside container>
      - '3306:3306'
    expose:
      # Opens port 3306 on the container
      - '3306'
      # Where our data will be persisted
    volumes:
      - my-db:/var/lib/mysql
```

We just have to choose a platform to install the application in production.  
For the begginers i suggest the easyest orchestrator on the market [Docker Swarm](https://docs.docker.com/engine/swarm/)  
After a learning period it is possible to move towards on [Kubernetes](https://kubernetes.io/)  
Unlike Swarm you need to use a distribution, you can start with [k3s](https://k3s.io/), for the installation you can follow this repository [k3sup](https://github.com/alexellis/k3sup).  
There are a lot of kubernetes distribution avaliable on the cloud [AKS](https://azure.microsoft.com/en-us/services/kubernetes-service/), [EKS](https://aws.amazon.com/it/eks/), [GKE](https://cloud.google.com/kubernetes-engine); you just have to choose.

