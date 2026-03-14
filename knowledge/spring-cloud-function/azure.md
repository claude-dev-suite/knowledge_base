# Spring Cloud Function - Azure Functions

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-azure</artifactId>
</dependency>
<dependency>
    <groupId>com.microsoft.azure.functions</groupId>
    <artifactId>azure-functions-java-library</artifactId>
    <version>3.0.0</version>
</dependency>
```

## Handler Implementation

### HTTP Trigger Handler
```java
public class OrderHandler extends FunctionInvoker<HttpRequestMessage<String>, HttpResponseMessage> {

    @FunctionName("processOrder")
    public HttpResponseMessage run(
            @HttpTrigger(
                name = "req",
                methods = {HttpMethod.POST},
                authLevel = AuthorizationLevel.ANONYMOUS,
                route = "orders"
            ) HttpRequestMessage<String> request,
            ExecutionContext context) {

        return handleRequest(request, context);
    }
}
```

### Queue Trigger Handler
```java
public class QueueHandler extends FunctionInvoker<String, Void> {

    @FunctionName("processQueueMessage")
    public void run(
            @QueueTrigger(
                name = "message",
                queueName = "orders-queue",
                connection = "AzureWebJobsStorage"
            ) String message,
            ExecutionContext context) {

        handleRequest(message, context);
    }
}
```

### Blob Trigger Handler
```java
public class BlobHandler extends FunctionInvoker<byte[], Void> {

    @FunctionName("processBlobUpload")
    public void run(
            @BlobTrigger(
                name = "content",
                path = "uploads/{name}",
                connection = "AzureWebJobsStorage"
            ) byte[] content,
            @BindingName("name") String fileName,
            ExecutionContext context) {

        context.getLogger().info("Processing file: " + fileName);
        handleRequest(content, context);
    }
}
```

## Function Implementation

### Order Processing Function
```java
@Configuration
public class FunctionConfig {

    @Bean
    public Function<HttpRequestMessage<String>, HttpResponseMessage> processOrder(
            ObjectMapper objectMapper) {

        return request -> {
            try {
                String body = request.getBody();
                CreateOrderRequest orderRequest = objectMapper.readValue(
                    body, CreateOrderRequest.class);

                Order order = orderService.create(orderRequest);

                return request.createResponseBuilder(HttpStatus.CREATED)
                    .header("Content-Type", "application/json")
                    .body(objectMapper.writeValueAsString(order))
                    .build();

            } catch (ValidationException e) {
                return request.createResponseBuilder(HttpStatus.BAD_REQUEST)
                    .body("{\"error\": \"" + e.getMessage() + "\"}")
                    .build();

            } catch (Exception e) {
                return request.createResponseBuilder(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("{\"error\": \"Internal server error\"}")
                    .build();
            }
        };
    }

    @Bean
    public Consumer<String> processQueueMessage(ObjectMapper objectMapper) {
        return message -> {
            try {
                OrderEvent event = objectMapper.readValue(message, OrderEvent.class);
                orderProcessor.process(event);
            } catch (Exception e) {
                log.error("Error processing queue message", e);
                throw new RuntimeException(e);
            }
        };
    }
}
```

## Configuration

### application.yml
```yaml
spring:
  cloud:
    function:
      definition: processOrder

logging:
  level:
    root: INFO
    com.example: DEBUG
```

### host.json
```json
{
  "version": "2.0",
  "functionTimeout": "00:05:00",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[3.*, 4.0.0)"
  }
}
```

### local.settings.json
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "java",
    "SPRING_CLOUD_FUNCTION_DEFINITION": "processOrder"
  }
}
```

## Multiple Triggers

```java
@FunctionName("httpHandler")
public HttpResponseMessage httpTrigger(
        @HttpTrigger(
            name = "req",
            methods = {HttpMethod.GET, HttpMethod.POST},
            authLevel = AuthorizationLevel.FUNCTION,
            route = "api/{resource}"
        ) HttpRequestMessage<String> request,
        @BindingName("resource") String resource,
        ExecutionContext context) {

    context.getLogger().info("HTTP request for resource: " + resource);
    return handleRequest(request, context);
}

@FunctionName("timerHandler")
public void timerTrigger(
        @TimerTrigger(
            name = "timerInfo",
            schedule = "0 */5 * * * *"  // Every 5 minutes
        ) String timerInfo,
        ExecutionContext context) {

    context.getLogger().info("Timer trigger fired: " + timerInfo);
    handleScheduledTask();
}

@FunctionName("eventHubHandler")
public void eventHubTrigger(
        @EventHubTrigger(
            name = "events",
            eventHubName = "orders-hub",
            connection = "EventHubConnection",
            cardinality = Cardinality.MANY
        ) List<String> events,
        ExecutionContext context) {

    context.getLogger().info("Received " + events.size() + " events");
    events.forEach(this::processEvent);
}

@FunctionName("cosmosDbHandler")
public void cosmosDbTrigger(
        @CosmosDBTrigger(
            name = "documents",
            databaseName = "ordersdb",
            containerName = "orders",
            connection = "CosmosDbConnection",
            createLeaseContainerIfNotExists = true
        ) String[] documents,
        ExecutionContext context) {

    for (String document : documents) {
        processDocument(document);
    }
}
```

## Output Bindings

```java
@FunctionName("processAndNotify")
public void processAndNotify(
        @HttpTrigger(
            name = "req",
            methods = {HttpMethod.POST},
            authLevel = AuthorizationLevel.ANONYMOUS
        ) HttpRequestMessage<String> request,
        @QueueOutput(
            name = "notifications",
            queueName = "notification-queue",
            connection = "AzureWebJobsStorage"
        ) OutputBinding<String> notificationQueue,
        ExecutionContext context) {

    String body = request.getBody();
    // Process request

    // Send notification to queue
    notificationQueue.setValue("{\"message\": \"Order processed\"}");
}
```

## Maven Configuration

### pom.xml
```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.microsoft.azure</groupId>
            <artifactId>azure-functions-maven-plugin</artifactId>
            <version>1.24.0</version>
            <configuration>
                <appName>my-spring-function</appName>
                <resourceGroup>my-resource-group</resourceGroup>
                <appServicePlanName>my-plan</appServicePlanName>
                <region>eastus</region>
                <runtime>
                    <os>linux</os>
                    <javaVersion>17</javaVersion>
                </runtime>
                <appSettings>
                    <property>
                        <name>FUNCTIONS_EXTENSION_VERSION</name>
                        <value>~4</value>
                    </property>
                    <property>
                        <name>SPRING_CLOUD_FUNCTION_DEFINITION</name>
                        <value>processOrder</value>
                    </property>
                </appSettings>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## Deployment

```bash
# Build
mvn clean package

# Run locally
mvn azure-functions:run

# Deploy
mvn azure-functions:deploy

# View logs
az functionapp logs show --name my-spring-function --resource-group my-resource-group
```

## Best Practices

| Do | Don't |
|----|-------|
| Use appropriate trigger types | HTTP trigger for everything |
| Configure scaling settings | Default scaling only |
| Use Application Insights | Skip monitoring |
| Handle cold starts | Ignore startup time |
| Use managed identities | Store credentials in config |
| Configure function timeout | Use default timeout |
