# Spring AI - Chat Models

## Overview

Spring AI provides a unified API for interacting with various chat models including OpenAI, Anthropic Claude, Azure OpenAI, and more.

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
```

### Anthropic Claude
```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-opus-20240229
          temperature: 0.7
          max-tokens: 4096
```

### Azure OpenAI
```yaml
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: ${AZURE_OPENAI_ENDPOINT}
        chat:
          options:
            deployment-name: gpt-4
            temperature: 0.7
```

## ChatClient API

### Basic Usage
```java
@Service
@RequiredArgsConstructor
public class ChatService {

    private final ChatClient chatClient;

    public String chat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
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

### ChatClient Builder
```java
@Configuration
public class ChatClientConfig {

    @Bean
    public ChatClient chatClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
            .defaultSystem("You are a helpful assistant specialized in software development.")
            .defaultOptions(ChatOptionsBuilder.builder()
                .withTemperature(0.7f)
                .withMaxTokens(2048)
                .build())
            .build();
    }
}
```

### Conversation History
```java
@Service
public class ConversationService {

    private final ChatClient chatClient;

    public String continueConversation(List<Message> history, String newMessage) {
        return chatClient.prompt()
            .messages(history)
            .user(newMessage)
            .call()
            .content();
    }
}

// Usage
List<Message> history = new ArrayList<>();
history.add(new UserMessage("What is Spring Boot?"));
history.add(new AssistantMessage("Spring Boot is a framework..."));

String response = conversationService.continueConversation(
    history, "How do I create a REST API with it?");
```

## Structured Output

### Entity Extraction
```java
public record ProductReview(
    String productName,
    int rating,
    String sentiment,
    List<String> pros,
    List<String> cons
) {}

@Service
public class ReviewAnalyzer {

    private final ChatClient chatClient;

    public ProductReview analyzeReview(String reviewText) {
        return chatClient.prompt()
            .user("Analyze this product review and extract structured data: " + reviewText)
            .call()
            .entity(ProductReview.class);
    }
}
```

### List Extraction
```java
@Service
public class TopicExtractor {

    private final ChatClient chatClient;

    public List<String> extractTopics(String document) {
        return chatClient.prompt()
            .user("Extract the main topics from this document: " + document)
            .call()
            .entity(new ParameterizedTypeReference<List<String>>() {});
    }
}
```

## Streaming Responses

### Basic Streaming
```java
@Service
public class StreamingChatService {

    private final ChatClient chatClient;

    public Flux<String> streamChat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .stream()
            .content();
    }
}

@RestController
public class ChatController {

    private final StreamingChatService chatService;

    @GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String message) {
        return chatService.streamChat(message);
    }
}
```

### Streaming with Metadata
```java
public Flux<ChatResponse> streamWithMetadata(String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .chatResponse();
}
```

## Function Calling

### Define Functions
```java
@Configuration
public class FunctionConfig {

    @Bean
    @Description("Get the current weather for a location")
    public Function<WeatherRequest, WeatherResponse> currentWeather() {
        return request -> {
            // Call weather API
            return weatherService.getWeather(request.location());
        };
    }

    @Bean
    @Description("Search for products by name or category")
    public Function<ProductSearchRequest, List<Product>> searchProducts() {
        return request -> productService.search(request.query(), request.category());
    }
}

public record WeatherRequest(String location) {}
public record WeatherResponse(String location, double temperature, String condition) {}
public record ProductSearchRequest(String query, String category) {}
```

### Use Functions in Chat
```java
@Service
public class FunctionCallingService {

    private final ChatClient chatClient;

    public String chatWithFunctions(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .functions("currentWeather", "searchProducts")
            .call()
            .content();
    }
}

// The model will automatically call functions when needed
// "What's the weather in Tokyo?" -> calls currentWeather function
// "Find me some laptops under $1000" -> calls searchProducts function
```

## Prompt Templates

### Using PromptTemplate
```java
@Service
public class TemplatedChatService {

    private final ChatClient chatClient;

    public String generateProductDescription(Product product) {
        PromptTemplate template = new PromptTemplate("""
            Create a compelling product description for the following item:

            Name: {name}
            Category: {category}
            Features: {features}
            Price: {price}

            The description should be engaging and highlight the key benefits.
            """);

        Prompt prompt = template.create(Map.of(
            "name", product.getName(),
            "category", product.getCategory(),
            "features", String.join(", ", product.getFeatures()),
            "price", product.getPrice().toString()
        ));

        return chatClient.prompt(prompt).call().content();
    }
}
```

### Resource-based Templates
```java
@Service
public class ResourceTemplateService {

    private final ChatClient chatClient;

    @Value("classpath:/prompts/code-review.st")
    private Resource codeReviewPrompt;

    public String reviewCode(String code, String language) {
        PromptTemplate template = new PromptTemplate(codeReviewPrompt);

        Prompt prompt = template.create(Map.of(
            "code", code,
            "language", language
        ));

        return chatClient.prompt(prompt).call().content();
    }
}
```

## Advisors (Interceptors)

```java
@Configuration
public class ChatAdvisorConfig {

    @Bean
    public ChatClient chatClient(ChatModel model) {
        return ChatClient.builder(model)
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(new InMemoryChatMemory()),
                new LoggingAdvisor(),
                new SafetyAdvisor()
            )
            .build();
    }
}

public class LoggingAdvisor implements RequestResponseAdvisor {

    @Override
    public AdvisedRequest adviseRequest(AdvisedRequest request, Map<String, Object> context) {
        log.info("Chat request: {}", request.userText());
        return request;
    }

    @Override
    public ChatResponse adviseResponse(ChatResponse response, Map<String, Object> context) {
        log.info("Chat response: {} tokens", response.getMetadata().getUsage().getTotalTokens());
        return response;
    }
}
```

## Error Handling

```java
@Service
@Slf4j
public class ResilientChatService {

    private final ChatClient chatClient;

    @Retryable(
        retryFor = {ChatClientException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String chatWithRetry(String message) {
        try {
            return chatClient.prompt()
                .user(message)
                .call()
                .content();
        } catch (ChatClientException e) {
            log.error("Chat API error: {}", e.getMessage());
            throw e;
        }
    }

    @Recover
    public String fallback(ChatClientException e, String message) {
        log.warn("Using fallback response after retries exhausted");
        return "I apologize, but I'm currently unable to process your request.";
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use system prompts for context | Rely only on user messages |
| Implement streaming for long responses | Block on long completions |
| Use structured output for data extraction | Parse unstructured responses |
| Configure appropriate temperature | Use defaults blindly |
| Implement retry logic | Assume API always succeeds |
| Use function calling for actions | Text-based command parsing |

## Production Checklist

- [ ] API keys secured in environment
- [ ] Rate limiting implemented
- [ ] Retry logic configured
- [ ] Streaming for UI responses
- [ ] Logging and monitoring
- [ ] Cost tracking enabled
- [ ] Content moderation considered
