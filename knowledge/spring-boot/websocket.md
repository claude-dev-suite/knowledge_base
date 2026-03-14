# Spring WebSocket Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

## STOMP Configuration

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins("http://localhost:3000")
            .withSockJS();
    }
}
```

## Message Controller

```java
@Controller
@RequiredArgsConstructor
public class ChatController {

    private final SimpMessagingTemplate messagingTemplate;

    // Broadcast to all subscribers
    @MessageMapping("/chat.send")
    @SendTo("/topic/messages")
    public ChatMessage sendMessage(@Payload ChatMessage message) {
        message.setTimestamp(Instant.now());
        return message;
    }

    // Send to specific user
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(@Payload PrivateMessage message,
                                   Principal principal) {
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),
            "/queue/private",
            message
        );
    }

    // With headers
    @MessageMapping("/chat.room")
    public void sendToRoom(@Payload ChatMessage message,
                          @Header("roomId") String roomId) {
        messagingTemplate.convertAndSend("/topic/room/" + roomId, message);
    }
}
```

## Send from Service

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final SimpMessagingTemplate messagingTemplate;

    public void broadcastNotification(Notification notification) {
        messagingTemplate.convertAndSend("/topic/notifications", notification);
    }

    public void sendToUser(String username, Notification notification) {
        messagingTemplate.convertAndSendToUser(
            username,
            "/queue/notifications",
            notification
        );
    }

    public void sendToRoom(String roomId, ChatMessage message) {
        messagingTemplate.convertAndSend("/topic/room/" + roomId, message);
    }
}
```

## Event Listeners

```java
@Component
@RequiredArgsConstructor
public class WebSocketEventListener {

    private final SimpMessagingTemplate messagingTemplate;

    @EventListener
    public void handleConnect(SessionConnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = accessor.getUser().getName();
        log.info("User connected: {}", username);
    }

    @EventListener
    public void handleDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String username = accessor.getUser().getName();
        log.info("User disconnected: {}", username);

        // Notify others
        messagingTemplate.convertAndSend("/topic/users",
            new UserEvent(username, "DISCONNECTED"));
    }

    @EventListener
    public void handleSubscribe(SessionSubscribeEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        log.info("User subscribed to: {}", accessor.getDestination());
    }
}
```

## Security

```java
@Configuration
@EnableWebSocketSecurity
public class WebSocketSecurityConfig {

    @Bean
    public AuthorizationManager<Message<?>> messageAuthorizationManager() {
        return MessageMatcherDelegatingAuthorizationManager.builder()
            .nullDestMatcher().authenticated()
            .simpDestMatchers("/app/**").authenticated()
            .simpSubscribeDestMatchers("/topic/**").authenticated()
            .simpSubscribeDestMatchers("/user/**").authenticated()
            .anyMessage().denyAll()
            .build();
    }
}

// JWT Authentication
@Configuration
public class WebSocketAuthConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);

                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    String token = accessor.getFirstNativeHeader("Authorization");
                    if (token != null && token.startsWith("Bearer ")) {
                        Authentication auth = validateToken(token.substring(7));
                        accessor.setUser(auth);
                    }
                }
                return message;
            }
        });
    }
}
```

## Client (JavaScript)

```javascript
import SockJS from 'sockjs-client';
import { Client } from '@stomp/stompjs';

const client = new Client({
  webSocketFactory: () => new SockJS('http://localhost:8080/ws'),
  connectHeaders: {
    Authorization: `Bearer ${token}`,
  },
  onConnect: () => {
    // Subscribe to topics
    client.subscribe('/topic/messages', (message) => {
      const body = JSON.parse(message.body);
      console.log('Received:', body);
    });

    // Subscribe to user-specific queue
    client.subscribe('/user/queue/notifications', (message) => {
      const body = JSON.parse(message.body);
      console.log('Private notification:', body);
    });
  },
  onDisconnect: () => {
    console.log('Disconnected');
  },
});

client.activate();

// Send message
function sendMessage(content) {
  client.publish({
    destination: '/app/chat.send',
    body: JSON.stringify({ content, sender: 'user1' }),
  });
}

// Send with headers
function sendToRoom(roomId, content) {
  client.publish({
    destination: '/app/chat.room',
    headers: { roomId },
    body: JSON.stringify({ content }),
  });
}
```

## Session Handling

```java
@Component
public class WebSocketSessionManager {

    private final Map<String, Set<String>> userSessions = new ConcurrentHashMap<>();

    public void addSession(String username, String sessionId) {
        userSessions.computeIfAbsent(username, k -> ConcurrentHashMap.newKeySet())
            .add(sessionId);
    }

    public void removeSession(String username, String sessionId) {
        Set<String> sessions = userSessions.get(username);
        if (sessions != null) {
            sessions.remove(sessionId);
            if (sessions.isEmpty()) {
                userSessions.remove(username);
            }
        }
    }

    public boolean isUserOnline(String username) {
        Set<String> sessions = userSessions.get(username);
        return sessions != null && !sessions.isEmpty();
    }

    public Set<String> getOnlineUsers() {
        return userSessions.keySet();
    }
}
```

## Message DTOs

```java
public record ChatMessage(
    String sender,
    String content,
    Instant timestamp,
    MessageType type
) {
    public enum MessageType { CHAT, JOIN, LEAVE }
}

public record PrivateMessage(
    String sender,
    String recipient,
    String content,
    Instant timestamp
) {}

public record Notification(
    String id,
    String message,
    NotificationType type,
    Instant timestamp
) {
    public enum NotificationType { INFO, WARNING, ERROR }
}
```

## Error Handling

```java
@ControllerAdvice
public class WebSocketExceptionHandler {

    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public ErrorMessage handleException(Exception ex, Principal principal) {
        log.error("WebSocket error for user {}: {}", principal.getName(), ex.getMessage());
        return new ErrorMessage(ex.getMessage());
    }
}
```
