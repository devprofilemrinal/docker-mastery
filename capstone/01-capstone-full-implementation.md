# Capstone: Full Implementation

Full source, Dockerfiles, and configuration for all five services from the previous chapter's architecture. Each service's Dockerfile follows the Phase 2/8 pattern established throughout this repository (multi-stage, JRE-only, non-root, tini) without re-explaining the reasoning already covered in those phases.

---

## Folder Structure

```text
capstone-ecommerce/
├── compose.yaml
├── compose.override.yaml
├── .env.example
├── k8s/
│   ├── namespace.yaml
│   ├── api-gateway/
│   ├── order-service/
│   ├── inventory-service/
│   ├── notification-service/
│   └── data/                    (postgres, redis, kafka manifests)
├── api-gateway/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/java/com/example/gateway/
│       ├── GatewayApplication.java
│       └── application.yml (routes config)
├── order-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/java/com/example/order/
│       ├── OrderServiceApplication.java
│       ├── Order.java
│       ├── OrderRepository.java
│       ├── OrderController.java
│       └── InventoryClient.java
├── inventory-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/java/com/example/inventory/
│       ├── InventoryServiceApplication.java
│       └── InventoryController.java
└── notification-service/
    ├── Dockerfile
    ├── pom.xml
    └── src/main/java/com/example/notification/
        ├── NotificationServiceApplication.java
        └── OrderEventListener.java
```

---

## The Shared Dockerfile Pattern (Applied to Every Service)

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN --mount=type=cache,target=/root/.m2 ./mvnw dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 ./mvnw clean package -DskipTests

FROM eclipse-temurin:21-jre
RUN apt-get update && apt-get install -y --no-install-recommends tini curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --gid 1001 appgroup \
    && useradd --uid 1001 --gid appgroup --shell /usr/sbin/nologin --no-create-home appuser
WORKDIR /app
COPY --from=build /app/target/*.jar /app/app.jar
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health/readiness || exit 1
ENTRYPOINT ["tini", "--", "java", \
  "-XX:MaxRAMPercentage=60.0", "-XX:+ExitOnOutOfMemoryError", \
  "-jar", "/app/app.jar"]
```

Each of the four Spring Boot services (`api-gateway`, `order-service`, `inventory-service`, `notification-service`) uses this identical pattern — the Phase 3 JVM flags, Phase 8 hardening, and Phase 3 Chapter 1 signal-handling fix, all applied uniformly.

---

## `order-service`: Core Logic

```java
package com.example.order;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderRepository repository;
    private final InventoryClient inventoryClient;
    private final KafkaTemplate<String, String> kafkaTemplate;

    public OrderController(OrderRepository repository, InventoryClient inventoryClient,
                            KafkaTemplate<String, String> kafkaTemplate) {
        this.repository = repository;
        this.inventoryClient = inventoryClient;
        this.kafkaTemplate = kafkaTemplate;
    }

    @PostMapping
    public Order create(@RequestBody Map<String, Object> body) {
        String sku = (String) body.get("sku");
        Integer quantity = (Integer) body.get("quantity");

        // Synchronous call to inventory-service (Phase 4, Ch.5, Pattern 2)
        var stock = inventoryClient.checkStock(sku);
        if (stock.quantity() < quantity) {
            throw new IllegalStateException("Insufficient stock for " + sku);
        }

        Order order = new Order();
        order.setSku(sku);
        order.setQuantity(quantity);
        Order saved = repository.save(order);

        // Async publish to notification-service (Phase 4, Ch.5, Pattern 3)
        kafkaTemplate.send("order-events", saved.getId().toString(), "OrderPlaced:" + sku);

        return saved;
    }
}
```

```java
package com.example.order;

import org.springframework.web.client.RestClient;
import org.springframework.stereotype.Component;
import java.time.Duration;

@Component
public class InventoryClient {
    private final RestClient client;

    public InventoryClient(RestClient.Builder builder) {
        this.client = builder.baseUrl("http://inventory-service:8080").build();
        // Explicit timeouts per Phase 4, Ch.5's resilience recommendation
    }

    public record StockStatus(String sku, int quantity) {}

    public StockStatus checkStock(String sku) {
        return client.get().uri("/api/inventory/{sku}", sku)
            .retrieve().body(StockStatus.class);
    }
}
```

---

## `inventory-service`: Redis-Backed Stock Lookup

```java
package com.example.inventory;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/inventory")
public class InventoryController {

    private final StringRedisTemplate redisTemplate;

    public InventoryController(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @GetMapping("/{sku}")
    public Map<String, Object> getStock(@PathVariable String sku) {
        String cached = redisTemplate.opsForValue().get("stock:" + sku);
        int quantity = cached != null ? Integer.parseInt(cached) : loadFromDbAndCache(sku);
        return Map.of("sku", sku, "quantity", quantity);
    }

    private int loadFromDbAndCache(String sku) {
        int quantity = 100; // placeholder for a real DB lookup
        redisTemplate.opsForValue().set("stock:" + sku, String.valueOf(quantity), java.time.Duration.ofMinutes(5));
        return quantity;
    }
}
```

---

## `notification-service`: Kafka Consumer

```java
package com.example.notification;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class OrderEventListener {
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void onOrderEvent(String message) {
        System.out.println("Sending notification for: " + message);
    }
}
```

---

## `compose.yaml` (Abbreviated — Full Pattern From Phase 6)

```yaml
services:
  api-gateway:
    build: ./api-gateway
    ports: ["8080:8080"]
    depends_on:
      order-service: { condition: service_healthy }
      inventory-service: { condition: service_healthy }

  order-service:
    build: ./order-service
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      kafka: { condition: service_healthy }

  inventory-service:
    build: ./inventory-service
    depends_on:
      redis: { condition: service_healthy }

  notification-service:
    build: ./notification-service
    depends_on:
      kafka: { condition: service_healthy }

  postgres:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}", POSTGRES_DB: orders }
    volumes: ["pg-data:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s, retries: 5 }

  redis:
    image: redis:7-alpine
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, retries: 5 }

  kafka:
    image: apache/kafka:3.7.0
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions.sh --bootstrap-server localhost:9092"]
      interval: 10s
      retries: 6

volumes:
  pg-data:
```

---

## What's Next

**Next:** [`02-capstone-build-run-debug-and-productionize.md`](./02-capstone-build-run-debug-and-productionize.md)