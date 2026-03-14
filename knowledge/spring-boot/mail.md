# Spring Mail Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

## Configuration

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 5000
          writetimeout: 5000
```

## Simple Email

```java
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;

    public void sendSimpleEmail(String to, String subject, String text) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom("noreply@example.com");
        message.setTo(to);
        message.setSubject(subject);
        message.setText(text);

        mailSender.send(message);
    }

    public void sendToMultiple(String[] to, String subject, String text) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom("noreply@example.com");
        message.setTo(to);
        message.setCc("manager@example.com");
        message.setBcc("audit@example.com");
        message.setSubject(subject);
        message.setText(text);

        mailSender.send(message);
    }
}
```

## HTML Email

```java
@Service
@RequiredArgsConstructor
public class HtmlEmailService {

    private final JavaMailSender mailSender;

    public void sendHtmlEmail(String to, String subject, String htmlContent)
            throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

        helper.setFrom("noreply@example.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlContent, true);  // true = HTML

        mailSender.send(message);
    }
}
```

## Email with Attachments

```java
public void sendEmailWithAttachment(String to, String subject, String text,
                                    String attachmentPath) throws MessagingException {
    MimeMessage message = mailSender.createMimeMessage();
    MimeMessageHelper helper = new MimeMessageHelper(message, true);

    helper.setFrom("noreply@example.com");
    helper.setTo(to);
    helper.setSubject(subject);
    helper.setText(text);

    // File attachment
    FileSystemResource file = new FileSystemResource(new File(attachmentPath));
    helper.addAttachment(file.getFilename(), file);

    // Inline image
    helper.addInline("logo", new ClassPathResource("images/logo.png"));

    mailSender.send(message);
}
```

## Thymeleaf Templates

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

```html
<!-- templates/email/welcome.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
</head>
<body>
    <h1>Welcome, <span th:text="${name}">User</span>!</h1>
    <p>Thank you for registering.</p>
    <a th:href="${activationLink}">Activate your account</a>
</body>
</html>
```

```java
@Service
@RequiredArgsConstructor
public class TemplateEmailService {

    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;

    public void sendWelcomeEmail(String to, String name, String activationLink)
            throws MessagingException {
        Context context = new Context();
        context.setVariable("name", name);
        context.setVariable("activationLink", activationLink);

        String htmlContent = templateEngine.process("email/welcome", context);

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

        helper.setFrom("noreply@example.com");
        helper.setTo(to);
        helper.setSubject("Welcome to Our Platform!");
        helper.setText(htmlContent, true);

        mailSender.send(message);
    }
}
```

## Async Email

```java
@Service
@RequiredArgsConstructor
public class AsyncEmailService {

    private final JavaMailSender mailSender;

    @Async
    public CompletableFuture<Void> sendEmailAsync(String to, String subject, String text) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(to);
            message.setSubject(subject);
            message.setText(text);
            mailSender.send(message);
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

## Email Queue with Events

```java
public record SendEmailEvent(
    String to,
    String subject,
    String template,
    Map<String, Object> variables
) {}

@Component
@RequiredArgsConstructor
public class EmailEventListener {

    private final TemplateEmailService emailService;

    @Async
    @EventListener
    public void handleEmailEvent(SendEmailEvent event) {
        try {
            emailService.sendTemplatedEmail(
                event.to(),
                event.subject(),
                event.template(),
                event.variables()
            );
        } catch (Exception e) {
            log.error("Failed to send email to {}: {}", event.to(), e.getMessage());
        }
    }
}

// Usage
@Service
@RequiredArgsConstructor
public class UserService {

    private final ApplicationEventPublisher eventPublisher;

    public void registerUser(RegisterRequest request) {
        User user = createUser(request);

        eventPublisher.publishEvent(new SendEmailEvent(
            user.getEmail(),
            "Welcome!",
            "email/welcome",
            Map.of("name", user.getName(), "link", generateActivationLink(user))
        ));
    }
}
```

## Email Builder Pattern

```java
@Builder
public record Email(
    String from,
    List<String> to,
    List<String> cc,
    List<String> bcc,
    String subject,
    String text,
    boolean html,
    List<Attachment> attachments
) {
    @Builder
    public record Attachment(String name, Resource resource) {}
}

@Service
@RequiredArgsConstructor
public class EmailSenderService {

    private final JavaMailSender mailSender;

    public void send(Email email) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

        helper.setFrom(email.from());
        helper.setTo(email.to().toArray(new String[0]));
        if (email.cc() != null) helper.setCc(email.cc().toArray(new String[0]));
        if (email.bcc() != null) helper.setBcc(email.bcc().toArray(new String[0]));
        helper.setSubject(email.subject());
        helper.setText(email.text(), email.html());

        for (Email.Attachment attachment : email.attachments()) {
            helper.addAttachment(attachment.name(), attachment.resource());
        }

        mailSender.send(message);
    }
}

// Usage
emailSender.send(Email.builder()
    .from("noreply@example.com")
    .to(List.of("user@example.com"))
    .subject("Your Report")
    .text(htmlContent)
    .html(true)
    .attachments(List.of(
        Email.Attachment.builder()
            .name("report.pdf")
            .resource(new ByteArrayResource(pdfBytes))
            .build()
    ))
    .build());
```

## Testing

```java
@SpringBootTest
class EmailServiceTest {

    @Autowired
    private EmailService emailService;

    @MockBean
    private JavaMailSender mailSender;

    @Test
    void shouldSendEmail() {
        emailService.sendSimpleEmail("test@example.com", "Subject", "Body");

        verify(mailSender).send(any(SimpleMailMessage.class));
    }
}
```
