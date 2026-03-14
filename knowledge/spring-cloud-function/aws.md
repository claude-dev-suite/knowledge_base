# Spring Cloud Function - AWS Lambda

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
</dependency>
```

## Handler Configuration

### Simple Handler
```java
public class LambdaHandler extends FunctionInvoker<String, String> {
}
```

### API Gateway Handler
```java
public class ApiHandler extends FunctionInvoker<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
}
```

## Function Implementation

### Basic Function
```java
@Bean
public Function<String, String> processMessage() {
    return message -> {
        log.info("Processing: {}", message);
        return "Processed: " + message;
    };
}
```

### API Gateway Function
```java
@Bean
public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> handleApiRequest() {
    return request -> {
        String body = request.getBody();
        Map<String, String> headers = request.getHeaders();
        Map<String, String> pathParams = request.getPathParameters();
        Map<String, String> queryParams = request.getQueryStringParameters();

        // Process request
        String result = processRequest(body, pathParams);

        return APIGatewayProxyResponseEvent.builder()
            .statusCode(200)
            .headers(Map.of("Content-Type", "application/json"))
            .body(result)
            .build();
    };
}

@Bean
public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> createOrder(
        ObjectMapper objectMapper) {
    return request -> {
        try {
            CreateOrderRequest orderRequest = objectMapper.readValue(
                request.getBody(), CreateOrderRequest.class);

            Order order = orderService.create(orderRequest);

            return APIGatewayProxyResponseEvent.builder()
                .statusCode(201)
                .headers(Map.of("Content-Type", "application/json"))
                .body(objectMapper.writeValueAsString(order))
                .build();

        } catch (ValidationException e) {
            return APIGatewayProxyResponseEvent.builder()
                .statusCode(400)
                .body("{\"error\": \"" + e.getMessage() + "\"}")
                .build();
        } catch (Exception e) {
            log.error("Error creating order", e);
            return APIGatewayProxyResponseEvent.builder()
                .statusCode(500)
                .body("{\"error\": \"Internal server error\"}")
                .build();
        }
    };
}
```

### SQS Event Handler
```java
@Bean
public Consumer<SQSEvent> processSqsMessages() {
    return sqsEvent -> {
        for (SQSEvent.SQSMessage message : sqsEvent.getRecords()) {
            log.info("Processing SQS message: {}", message.getMessageId());
            processMessage(message.getBody());
        }
    };
}
```

### S3 Event Handler
```java
@Bean
public Consumer<S3Event> processS3Event() {
    return s3Event -> {
        for (S3Event.S3EventNotificationRecord record : s3Event.getRecords()) {
            String bucket = record.getS3().getBucket().getName();
            String key = record.getS3().getObject().getKey();
            log.info("S3 event: s3://{}/{}", bucket, key);
            processFile(bucket, key);
        }
    };
}
```

## Configuration

### application.yml
```yaml
spring:
  cloud:
    function:
      definition: handleApiRequest

# AWS specific
aws:
  region: us-east-1

logging:
  level:
    root: INFO
    com.example: DEBUG
```

## SAM Template

### template.yaml
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: java17
    Architectures:
      - x86_64
    Environment:
      Variables:
        SPRING_PROFILES_ACTIVE: prod

Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: org.springframework.cloud.function.adapter.aws.FunctionInvoker::handleRequest
      CodeUri: target/order-function.jar
      MemorySize: 1024
      Timeout: 60
      Environment:
        Variables:
          SPRING_CLOUD_FUNCTION_DEFINITION: createOrder
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /orders
            Method: POST

  ProcessMessagesFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: org.springframework.cloud.function.adapter.aws.FunctionInvoker::handleRequest
      CodeUri: target/order-function.jar
      Environment:
        Variables:
          SPRING_CLOUD_FUNCTION_DEFINITION: processSqsMessages
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10

  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: orders-queue
      VisibilityTimeout: 120

Outputs:
  ApiEndpoint:
    Description: API Gateway endpoint
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

## Reducing Cold Start

### GraalVM Native Image
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <builder>paketobuildpacks/builder:tiny</builder>
            <env>
                <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
            </env>
        </image>
    </configuration>
</plugin>
```

### SnapStart
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      SnapStart:
        ApplyOn: PublishedVersions
```

### Class Data Sharing (CDS)
```dockerfile
FROM amazoncorretto:17-alpine
COPY target/function.jar /app/function.jar
RUN java -Xshare:dump -jar /app/function.jar --thin.dryrun
CMD ["java", "-Xshare:on", "-jar", "/app/function.jar"]
```

## Environment Variables

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  cloud:
    function:
      definition: ${FUNCTION_NAME:processOrder}
```

## Packaging

### Maven Plugin
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.4.1</version>
    <configuration>
        <createDependencyReducedPom>false</createDependencyReducedPom>
        <transformers>
            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                <resource>META-INF/spring.handlers</resource>
            </transformer>
            <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                <resource>META-INF/spring.schemas</resource>
            </transformer>
        </transformers>
    </configuration>
</plugin>
```

## Best Practices

| Do | Don't |
|----|-------|
| Use SnapStart or native image | Ignore cold start optimization |
| Configure appropriate memory | Under-provision memory |
| Handle timeouts gracefully | Assume infinite execution time |
| Use environment variables for config | Hardcode configuration |
| Test with SAM local | Deploy without local testing |
| Use structured logging | Log without context |

## Deployment Commands

```bash
# Build
mvn clean package

# Local testing
sam local invoke OrderFunction -e event.json

# Deploy
sam deploy --guided

# View logs
sam logs -n OrderFunction --tail
```
