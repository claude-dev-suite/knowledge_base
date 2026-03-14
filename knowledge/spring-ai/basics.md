# Spring AI - Basics

## Overview

Spring AI provides a consistent API for integrating AI capabilities into Spring applications. It supports multiple AI providers including OpenAI, Azure OpenAI, Anthropic, and more.

## Dependencies

### OpenAI
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

### Anthropic
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>
```

### Azure OpenAI
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
</dependency>
```

## Configuration

### OpenAI
```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 2000
```

### Anthropic
```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-opus-20240229
          temperature: 0.7
          max-tokens: 2000
```

## ChatClient

### Basic Usage
```java
@Service
@RequiredArgsConstructor
public class ChatService {

    private final ChatClient chatClient;

    public String chat(String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }

    public String chatWithSystem(String systemPrompt, String userMessage) {
        return chatClient.prompt()
            .system(systemPrompt)
            .user(userMessage)
            .call()
            .content();
    }
}
```

### With Parameters
```java
public String generateContent(String topic) {
    return chatClient.prompt()
        .system("""
            You are a technical writer. Write clear, concise content.
            Format: Markdown
            Tone: Professional
            """)
        .user("Write an article about: " + topic)
        .options(ChatOptionsBuilder.builder()
            .withTemperature(0.8f)
            .withMaxTokens(1500)
            .build())
        .call()
        .content();
}
```

### Streaming
```java
public Flux<String> chatStream(String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

## Prompt Templates

### Using PromptTemplate
```java
@Service
public class TemplateService {

    private final ChatClient chatClient;

    public String generateEmail(String recipient, String subject, String context) {
        PromptTemplate template = new PromptTemplate("""
            Write a professional email to {recipient} about {subject}.

            Context: {context}

            Requirements:
            - Professional tone
            - Clear and concise
            - Include greeting and sign-off
            """);

        Prompt prompt = template.create(Map.of(
            "recipient", recipient,
            "subject", subject,
            "context", context
        ));

        return chatClient.prompt(prompt)
            .call()
            .content();
    }
}
```

### Resource Templates
```java
@Service
public class ResourceTemplateService {

    @Value("classpath:/prompts/analysis.st")
    private Resource analysisPromptResource;

    private final ChatClient chatClient;

    public String analyze(String data) {
        PromptTemplate template = new PromptTemplate(analysisPromptResource);

        return chatClient.prompt(template.create(Map.of("data", data)))
            .call()
            .content();
    }
}
```

## Structured Output

### Using BeanOutputConverter
```java
public record ProductDescription(
    String title,
    String description,
    List<String> features,
    String targetAudience
) {}

@Service
public class ProductService {

    private final ChatClient chatClient;

    public ProductDescription generateProductDescription(String productName) {
        BeanOutputConverter<ProductDescription> converter =
            new BeanOutputConverter<>(ProductDescription.class);

        String response = chatClient.prompt()
            .system(converter.getFormat())
            .user("Generate a product description for: " + productName)
            .call()
            .content();

        return converter.convert(response);
    }
}
```

### Using entity() Method
```java
public ProductDescription generateDescription(String productName) {
    return chatClient.prompt()
        .user("Generate a product description for: " + productName)
        .call()
        .entity(ProductDescription.class);
}
```

## Function Calling

### Define Functions
```java
@Configuration
public class FunctionConfig {

    @Bean
    @Description("Get the current weather for a location")
    public Function<WeatherRequest, WeatherResponse> getWeather() {
        return request -> weatherService.getWeather(request.location());
    }

    @Bean
    @Description("Search for products by query")
    public Function<ProductSearchRequest, List<Product>> searchProducts() {
        return request -> productService.search(request.query());
    }
}

public record WeatherRequest(String location) {}
public record WeatherResponse(String location, double temperature, String condition) {}
public record ProductSearchRequest(String query) {}
```

### Use Functions
```java
public String chatWithFunctions(String message) {
    return chatClient.prompt()
        .user(message)
        .functions("getWeather", "searchProducts")
        .call()
        .content();
}
```

## Advisors

### QuestionAnswerAdvisor
```java
@Service
public class QaService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String answerQuestion(String question) {
        return chatClient.prompt()
            .user(question)
            .advisors(new QuestionAnswerAdvisor(vectorStore))
            .call()
            .content();
    }
}
```

### MessageChatMemoryAdvisor
```java
@Service
public class ConversationService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public String chat(String sessionId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(new MessageChatMemoryAdvisor(chatMemory, sessionId, 10))
            .call()
            .content();
    }
}
```

## Error Handling

```java
@Service
public class ResilientChatService {

    private final ChatClient chatClient;

    @Retryable(
        retryFor = {RateLimitException.class, ServiceUnavailableException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String chat(String message) {
        try {
            return chatClient.prompt()
                .user(message)
                .call()
                .content();
        } catch (Exception e) {
            log.error("Chat error", e);
            throw new ChatServiceException("Failed to process chat", e);
        }
    }

    @Recover
    public String recoverChat(Exception e, String message) {
        return "I'm sorry, I'm temporarily unavailable. Please try again later.";
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use system prompts for context | Rely on user messages only |
| Implement retry for rate limits | Fail immediately |
| Use structured output | Parse unstructured responses |
| Validate AI responses | Trust output blindly |
| Log prompts and responses | Skip observability |
| Cache repeated queries | Call API for same query |

## Production Checklist

- [ ] API keys secured
- [ ] Rate limiting configured
- [ ] Retry mechanism in place
- [ ] Response validation
- [ ] Logging and monitoring
- [ ] Cost monitoring
- [ ] Fallback strategies
