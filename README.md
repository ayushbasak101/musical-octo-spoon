
# Healthcare Microservices Architecture  
**Technical Configuration Guide**

---

## System Overview

This document provides comprehensive configuration and setup instructions for a distributed **Healthcare Management System** built using **Spring Boot Microservices Architecture**.  
The system follows a **loosely coupled, event-driven design** using:

- **gRPC** for synchronous service-to-service communication  
- **Apache Kafka** for asynchronous messaging and event streaming  
- **PostgreSQL** with a **database-per-service** pattern  

This architecture ensures **scalability, fault isolation, security, and maintainability**.

---

## Architecture Components

| Service | Description |
|--------|-------------|
| **Patient Service** | Core service managing patient records and medical data |
| **Billing Service** | Handles payment processing and invoice generation via gRPC |
| **Notification Service** | Processes asynchronous events via Kafka |
| **Authentication Service** | JWT-based authentication and authorization |
| **Message Broker** | Apache Kafka for inter-service communication |

---

## Patient Service Configuration

The **Patient Service** acts as the primary data management layer and integrates with:

- **Billing Service** via gRPC  
- **Notification Service** via Kafka  

### Environment Configuration

#### Database
```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_DATASOURCE_PASSWORD=password
````

#### JPA / Hibernate

```bash
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

#### Kafka

```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

#### gRPC Client

```bash
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9005
```

#### Debug (Development Only)

```bash
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

---

## gRPC Client Dependencies

Add to `pom.xml`:

```xml
<!-- gRPC Core -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.69.0</version>
</dependency>
```

Additional required dependencies:

* `grpc-protobuf`
* `grpc-stub`
* `protobuf-java`
* `net.devh:grpc-spring-boot-starter:3.1.0.RELEASE`

---

## Maven Build Configuration (Protobuf)

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
        </plugin>
    </plugins>
</build>
```

---

## Billing Service Configuration

The **Billing Service** exposes **gRPC endpoints** for:

* Payment processing
* Invoice generation

### Key Settings

* gRPC Server Port: `9005`
* Must be reachable via internal network
* Uses same Protobuf and gRPC dependencies as Patient Service

---

## Notification Service Configuration

Consumes Kafka topics and handles:

* Appointment reminders
* Billing notifications
* System alerts

### Environment

```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.3.0</version>
</dependency>
```

---

## Authentication Service Configuration

Provides **JWT-based security** for the entire system.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
```

### Database

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_DATASOURCE_PASSWORD=password
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

### Initial Admin User

`src/main/resources/data.sql`

```sql
CREATE TABLE IF NOT EXISTS "users" (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
);

INSERT INTO "users" (id, email, password, role)
SELECT '223e4567-e89b-12d3-a456-426614174006',
       'admin@system.com',
       '$2b$12$7hoRZfJrRKD2nIm2vHLs7OBETy...',
       'ADMIN'
WHERE NOT EXISTS (
    SELECT 1 FROM "users"
    WHERE email = 'admin@system.com'
);
```

---

## Apache Kafka Configuration (KRaft Mode)

Kafka runs **without ZooKeeper**.

### Broker Environment

```bash
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
```

### Kafka Consumer (Patient Service)

```properties
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
```

---

## PostgreSQL Configuration

Each service has its **own dedicated database**.

```bash
POSTGRES_DB=db
POSTGRES_USER=admin_user
POSTGRES_PASSWORD=password
```

> **Security Note:**
> Use secret managers in production:
>
> * HashiCorp Vault
> * AWS Secrets Manager
> * Kubernetes Secrets

---

## Deployment Architecture

The system is designed for **containerized deployment** using:

* Docker
* Docker Compose
* Kubernetes

Each service runs in its own container with isolated resources.

---

## Network Configuration

All services communicate over an internal Docker network using **service names as hostnames**.

Examples:

* `patient-service`
* `billing-service`
* `kafka`

---

## Port Mappings

| Service                | Port |
| ---------------------- | ---- |
| Patient Service (HTTP) | 8080 |
| Billing Service (gRPC) | 9005 |
| Authentication Service | 8081 |
| Kafka Internal         | 9092 |
| Kafka External         | 9094 |
| PostgreSQL             | 5432 |

---

## Production Best Practices

### Security

* API Gateway with rate limiting
* TLS for inter-service communication
* Rotate JWT secrets
* Encrypt databases at rest and in transit

### Monitoring & Observability

* Distributed tracing: Jaeger / Zipkin
* Centralized logging: ELK / Splunk
* Health checks & readiness probes
* Monitor Kafka consumer lag

### Scalability

* Horizontal scaling via Kubernetes
* Kafka partitions for parallel processing
* Database read replicas
* Distributed caching (Redis / Memcached)

---

## Architecture Principles

This system follows:

* **12-Factor App methodology**
* **Database-per-service pattern**
* **Event-driven microservices**
* **Strong service isolation**
* **Cloud-native design**

This ensures:

> **High availability, fault tolerance, horizontal scalability, and enterprise-grade security.**

```
```
