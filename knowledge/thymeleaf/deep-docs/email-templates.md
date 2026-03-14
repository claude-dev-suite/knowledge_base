# Thymeleaf Email Templates Guide

## Email Template Architecture

### Project Structure

```
src/main/resources/
├── templates/
│   └── email/
│       ├── user-registration.html
│       ├── password-reset.html
│       ├── order-confirmation.html
│       ├── invoice.html
│       └── newsletter.html
├── static/
│   └── email/
│       ├── images/
│       │   ├── logo.png
│       │   └── banner.jpg
│       └── css/
│           └── email-styles.css
└── application.yml
```

## Email Service Configuration

### Spring Boot Configuration

```yaml
# application.yml
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
        transport:
          protocol: smtp
    default-encoding: UTF-8

  thymeleaf:
    mode: HTML
    encoding: UTF-8
    cache: false  # Disable for development
    prefix: classpath:/templates/
    suffix: .html
```

### Email Service Implementation

```java
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;
    private final SpringTemplateEngine templateEngine;

    @Value("${app.mail.from}")
    private String fromEmail;

    @Value("${app.mail.from-name}")
    private String fromName;

    public void sendEmail(String to, String subject, String templateName, Map<String, Object> variables) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(
                message,
                MimeMessageHelper.MULTIPART_MODE_MIXED,
                StandardCharsets.UTF_8.name()
            );

            // Process template
            Context context = new Context();
            context.setVariables(variables);
            String html = templateEngine.process("email/" + templateName, context);

            // Configure message
            helper.setFrom(new InternetAddress(fromEmail, fromName));
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(html, true); // true = HTML

            // Send email
            mailSender.send(message);

            log.info("Email sent successfully to: {}", to);
        } catch (MessagingException | UnsupportedEncodingException e) {
            log.error("Failed to send email to: {}", to, e);
            throw new EmailSendException("Failed to send email", e);
        }
    }

    public void sendEmailWithAttachment(
        String to,
        String subject,
        String templateName,
        Map<String, Object> variables,
        String attachmentName,
        byte[] attachmentData
    ) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, StandardCharsets.UTF_8.name());

            Context context = new Context();
            context.setVariables(variables);
            String html = templateEngine.process("email/" + templateName, context);

            helper.setFrom(new InternetAddress(fromEmail, fromName));
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(html, true);

            // Add attachment
            helper.addAttachment(attachmentName, new ByteArrayResource(attachmentData));

            mailSender.send(message);
            log.info("Email with attachment sent successfully to: {}", to);
        } catch (MessagingException | UnsupportedEncodingException e) {
            log.error("Failed to send email with attachment to: {}", to, e);
            throw new EmailSendException("Failed to send email with attachment", e);
        }
    }

    public void sendEmailWithInlineImage(
        String to,
        String subject,
        String templateName,
        Map<String, Object> variables,
        String imageCid,
        byte[] imageData
    ) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, StandardCharsets.UTF_8.name());

            Context context = new Context();
            context.setVariables(variables);
            String html = templateEngine.process("email/" + templateName, context);

            helper.setFrom(new InternetAddress(fromEmail, fromName));
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(html, true);

            // Add inline image
            helper.addInline(imageCid, new ByteArrayResource(imageData), "image/png");

            mailSender.send(message);
        } catch (MessagingException | UnsupportedEncodingException e) {
            throw new EmailSendException("Failed to send email with inline image", e);
        }
    }
}
```

## Email Templates

### User Registration Email

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Our Platform</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        .header {
            background-color: #4A90E2;
            color: white;
            padding: 30px;
            text-align: center;
            border-radius: 5px 5px 0 0;
        }
        .content {
            background-color: #f9f9f9;
            padding: 30px;
            border-radius: 0 0 5px 5px;
        }
        .button {
            display: inline-block;
            padding: 12px 30px;
            background-color: #4A90E2;
            color: white;
            text-decoration: none;
            border-radius: 5px;
            margin: 20px 0;
        }
        .footer {
            text-align: center;
            margin-top: 20px;
            font-size: 12px;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>Welcome to Our Platform!</h1>
    </div>
    <div class="content">
        <p th:text="'Hello ' + ${userName} + ','">Hello User,</p>

        <p>Thank you for registering with us. We're excited to have you on board!</p>

        <p>Your account has been successfully created with the following details:</p>

        <ul>
            <li><strong>Email:</strong> <span th:text="${userEmail}">user@example.com</span></li>
            <li><strong>Registration Date:</strong> <span th:text="${#temporals.format(registrationDate, 'dd/MM/yyyy HH:mm')}">01/01/2024 10:00</span></li>
        </ul>

        <p>To get started, please verify your email address by clicking the button below:</p>

        <div style="text-align: center;">
            <a th:href="${verificationUrl}" class="button">Verify Email Address</a>
        </div>

        <p style="font-size: 12px; color: #666;">
            If the button doesn't work, copy and paste this link into your browser:<br>
            <span th:text="${verificationUrl}">https://example.com/verify?token=...</span>
        </p>

        <p>This verification link will expire in <span th:text="${expirationHours}">24</span> hours.</p>

        <p>If you didn't create this account, please ignore this email.</p>

        <p>Best regards,<br>The Team</p>
    </div>
    <div class="footer">
        <p>&copy; <span th:text="${#temporals.year(#temporals.createNow())}">2024</span> Your Company. All rights reserved.</p>
        <p>
            <a href="#" th:href="@{${baseUrl} + '/privacy'}">Privacy Policy</a> |
            <a href="#" th:href="@{${baseUrl} + '/terms'}">Terms of Service</a>
        </p>
    </div>
</body>
</html>
```

### Password Reset Email

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Password Reset Request</title>
    <style>
        /* Same base styles as above */
        .warning {
            background-color: #fff3cd;
            border-left: 4px solid #ffc107;
            padding: 15px;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>Password Reset Request</h1>
    </div>
    <div class="content">
        <p th:text="'Hello ' + ${userName} + ','">Hello User,</p>

        <p>We received a request to reset the password for your account.</p>

        <p>Click the button below to reset your password:</p>

        <div style="text-align: center;">
            <a th:href="${resetUrl}" class="button">Reset Password</a>
        </div>

        <p style="font-size: 12px; color: #666;">
            Or copy and paste this link:<br>
            <span th:text="${resetUrl}">https://example.com/reset-password?token=...</span>
        </p>

        <div class="warning">
            <strong>⚠️ Security Notice:</strong>
            <ul style="margin: 10px 0 0 0;">
                <li>This link will expire in <span th:text="${expirationMinutes}">30</span> minutes</li>
                <li>If you didn't request this reset, please ignore this email</li>
                <li>Your password will remain unchanged until you create a new one</li>
            </ul>
        </div>

        <p th:if="${loginAttempts != null and loginAttempts > 0}">
            <small>Recent failed login attempts: <strong th:text="${loginAttempts}">0</strong></small>
        </p>

        <p>Best regards,<br>The Security Team</p>
    </div>
    <div class="footer">
        <p>&copy; <span th:text="${#temporals.year(#temporals.createNow())}">2024</span> Your Company.</p>
    </div>
</body>
</html>
```

### Order Confirmation Email

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Order Confirmation</title>
    <style>
        /* Base styles */
        .order-summary {
            background-color: white;
            border: 1px solid #ddd;
            border-radius: 5px;
            padding: 20px;
            margin: 20px 0;
        }
        .order-item {
            display: flex;
            justify-content: space-between;
            padding: 10px 0;
            border-bottom: 1px solid #eee;
        }
        .total {
            font-size: 18px;
            font-weight: bold;
            margin-top: 15px;
            padding-top: 15px;
            border-top: 2px solid #333;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>Order Confirmation</h1>
    </div>
    <div class="content">
        <p th:text="'Hello ' + ${customerName} + ','">Hello Customer,</p>

        <p>Thank you for your order! We've received your order and it's being processed.</p>

        <div class="order-summary">
            <h2>Order Details</h2>
            <p><strong>Order Number:</strong> <span th:text="${orderNumber}">#12345</span></p>
            <p><strong>Order Date:</strong> <span th:text="${#temporals.format(orderDate, 'dd/MM/yyyy HH:mm')}">01/01/2024</span></p>
            <p><strong>Estimated Delivery:</strong> <span th:text="${#temporals.format(estimatedDelivery, 'dd/MM/yyyy')}">05/01/2024</span></p>

            <h3>Items</h3>
            <div th:each="item : ${orderItems}" class="order-item">
                <div>
                    <strong th:text="${item.name}">Product Name</strong><br>
                    <small>Quantity: <span th:text="${item.quantity}">1</span></small>
                </div>
                <div>
                    <span th:text="${#numbers.formatCurrency(item.price)}">€10.00</span>
                </div>
            </div>

            <div class="total">
                <div style="display: flex; justify-content: space-between;">
                    <span>Subtotal:</span>
                    <span th:text="${#numbers.formatCurrency(subtotal)}">€100.00</span>
                </div>
                <div style="display: flex; justify-content: space-between;">
                    <span>Shipping:</span>
                    <span th:text="${#numbers.formatCurrency(shippingCost)}">€5.00</span>
                </div>
                <div th:if="${discount > 0}" style="display: flex; justify-content: space-between; color: #28a745;">
                    <span>Discount:</span>
                    <span th:text="'-' + ${#numbers.formatCurrency(discount)}">-€10.00</span>
                </div>
                <div style="display: flex; justify-content: space-between; margin-top: 10px;">
                    <span>Total:</span>
                    <span th:text="${#numbers.formatCurrency(total)}">€95.00</span>
                </div>
            </div>
        </div>

        <div class="order-summary">
            <h3>Shipping Address</h3>
            <p th:text="${shippingAddress.fullName}">John Doe</p>
            <p th:text="${shippingAddress.street}">123 Main St</p>
            <p>
                <span th:text="${shippingAddress.city}">City</span>,
                <span th:text="${shippingAddress.zipCode}">12345</span>
            </p>
            <p th:text="${shippingAddress.country}">Country</p>
        </div>

        <div style="text-align: center;">
            <a th:href="@{${baseUrl} + '/orders/' + ${orderNumber}}" class="button">Track Your Order</a>
        </div>

        <p>You will receive another email when your order ships.</p>

        <p>Thank you for shopping with us!</p>
    </div>
    <div class="footer">
        <p>Questions? Contact us at <a th:href="'mailto:' + ${supportEmail}" th:text="${supportEmail}">support@example.com</a></p>
    </div>
</body>
</html>
```

### Invoice Email with PDF

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Invoice</title>
    <style>
        /* Professional invoice styling */
        .invoice-header {
            display: flex;
            justify-content: space-between;
            margin-bottom: 30px;
        }
        .company-info {
            text-align: left;
        }
        .invoice-info {
            text-align: right;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }
        th, td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        th {
            background-color: #f8f9fa;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="content">
        <div class="invoice-header">
            <div class="company-info">
                <h2 th:text="${companyName}">Company Name</h2>
                <p th:text="${companyAddress}">Company Address</p>
                <p>
                    <span th:text="${companyCity}">City</span>,
                    <span th:text="${companyZipCode}">ZIP</span>
                </p>
                <p>VAT: <span th:text="${companyVat}">IT12345678901</span></p>
            </div>
            <div class="invoice-info">
                <h2>INVOICE</h2>
                <p><strong>Invoice #:</strong> <span th:text="${invoiceNumber}">INV-2024-001</span></p>
                <p><strong>Date:</strong> <span th:text="${#temporals.format(invoiceDate, 'dd/MM/yyyy')}">01/01/2024</span></p>
                <p><strong>Due Date:</strong> <span th:text="${#temporals.format(dueDate, 'dd/MM/yyyy')}">31/01/2024</span></p>
            </div>
        </div>

        <div style="margin: 30px 0;">
            <h3>Bill To:</h3>
            <p><strong th:text="${customerName}">Customer Name</strong></p>
            <p th:text="${customerAddress}">Customer Address</p>
            <p th:if="${customerVat != null}">VAT: <span th:text="${customerVat}">VAT Number</span></p>
        </div>

        <table>
            <thead>
                <tr>
                    <th>Description</th>
                    <th style="text-align: center;">Quantity</th>
                    <th style="text-align: right;">Unit Price</th>
                    <th style="text-align: right;">Amount</th>
                </tr>
            </thead>
            <tbody>
                <tr th:each="item : ${invoiceItems}">
                    <td th:text="${item.description}">Item description</td>
                    <td style="text-align: center;" th:text="${item.quantity}">1</td>
                    <td style="text-align: right;" th:text="${#numbers.formatCurrency(item.unitPrice)}">€100.00</td>
                    <td style="text-align: right;" th:text="${#numbers.formatCurrency(item.amount)}">€100.00</td>
                </tr>
            </tbody>
            <tfoot>
                <tr>
                    <td colspan="3" style="text-align: right;"><strong>Subtotal:</strong></td>
                    <td style="text-align: right;" th:text="${#numbers.formatCurrency(subtotal)}">€1000.00</td>
                </tr>
                <tr>
                    <td colspan="3" style="text-align: right;">
                        <strong>VAT (<span th:text="${vatRate}">22</span>%):</strong>
                    </td>
                    <td style="text-align: right;" th:text="${#numbers.formatCurrency(vatAmount)}">€220.00</td>
                </tr>
                <tr>
                    <td colspan="3" style="text-align: right; font-size: 18px;"><strong>Total:</strong></td>
                    <td style="text-align: right; font-size: 18px; font-weight: bold;"
                        th:text="${#numbers.formatCurrency(total)}">€1220.00</td>
                </tr>
            </tfoot>
        </table>

        <div style="margin-top: 30px; padding: 15px; background-color: #f8f9fa; border-radius: 5px;">
            <h4>Payment Information</h4>
            <p><strong>Bank:</strong> <span th:text="${bankName}">Bank Name</span></p>
            <p><strong>IBAN:</strong> <span th:text="${iban}">IT60X0542811101000000123456</span></p>
            <p><strong>SWIFT/BIC:</strong> <span th:text="${swift}">ABCDIT12</span></p>
            <p><strong>Reference:</strong> <span th:text="${invoiceNumber}">INV-2024-001</span></p>
        </div>

        <p style="margin-top: 30px; font-size: 12px; color: #666;">
            Payment is due within <span th:text="${paymentTerms}">30</span> days.
            Late payments may incur additional charges.
        </p>

        <p style="text-align: center; margin-top: 40px;">
            <em>Thank you for your business!</em>
        </p>
    </div>
</body>
</html>
```

## Advanced Usage Examples

### Sending Emails with Service

```java
@Service
@RequiredArgsConstructor
public class UserNotificationService {

    private final EmailService emailService;
    private final UserRepository userRepository;

    @Value("${app.base-url}")
    private String baseUrl;

    public void sendWelcomeEmail(User user, String verificationToken) {
        Map<String, Object> variables = new HashMap<>();
        variables.put("userName", user.getName());
        variables.put("userEmail", user.getEmail());
        variables.put("registrationDate", user.getCreatedAt());
        variables.put("verificationUrl", baseUrl + "/verify?token=" + verificationToken);
        variables.put("expirationHours", 24);
        variables.put("baseUrl", baseUrl);

        emailService.sendEmail(
            user.getEmail(),
            "Welcome to Our Platform - Please Verify Your Email",
            "user-registration",
            variables
        );
    }

    public void sendPasswordResetEmail(User user, String resetToken) {
        Map<String, Object> variables = new HashMap<>();
        variables.put("userName", user.getName());
        variables.put("resetUrl", baseUrl + "/reset-password?token=" + resetToken);
        variables.put("expirationMinutes", 30);
        variables.put("loginAttempts", user.getFailedLoginAttempts());

        emailService.sendEmail(
            user.getEmail(),
            "Password Reset Request",
            "password-reset",
            variables
        );
    }

    public void sendOrderConfirmation(Order order) {
        Map<String, Object> variables = new HashMap<>();
        variables.put("customerName", order.getCustomer().getName());
        variables.put("orderNumber", order.getOrderNumber());
        variables.put("orderDate", order.getCreatedAt());
        variables.put("estimatedDelivery", order.getEstimatedDelivery());
        variables.put("orderItems", order.getItems());
        variables.put("subtotal", order.getSubtotal());
        variables.put("shippingCost", order.getShippingCost());
        variables.put("discount", order.getDiscount());
        variables.put("total", order.getTotal());
        variables.put("shippingAddress", order.getShippingAddress());
        variables.put("baseUrl", baseUrl);
        variables.put("supportEmail", "support@example.com");

        emailService.sendEmail(
            order.getCustomer().getEmail(),
            "Order Confirmation - " + order.getOrderNumber(),
            "order-confirmation",
            variables
        );
    }

    public void sendInvoiceEmail(Invoice invoice, byte[] pdfData) {
        Map<String, Object> variables = new HashMap<>();
        // Company info
        variables.put("companyName", "Your Company Ltd");
        variables.put("companyAddress", "123 Business St");
        variables.put("companyCity", "Milan");
        variables.put("companyZipCode", "20100");
        variables.put("companyVat", "IT12345678901");

        // Invoice info
        variables.put("invoiceNumber", invoice.getNumber());
        variables.put("invoiceDate", invoice.getDate());
        variables.put("dueDate", invoice.getDueDate());

        // Customer info
        variables.put("customerName", invoice.getCustomer().getName());
        variables.put("customerAddress", invoice.getCustomer().getAddress());
        variables.put("customerVat", invoice.getCustomer().getVatNumber());

        // Items and totals
        variables.put("invoiceItems", invoice.getItems());
        variables.put("subtotal", invoice.getSubtotal());
        variables.put("vatRate", invoice.getVatRate());
        variables.put("vatAmount", invoice.getVatAmount());
        variables.put("total", invoice.getTotal());

        // Payment info
        variables.put("bankName", "Your Bank");
        variables.put("iban", "IT60X0542811101000000123456");
        variables.put("swift", "ABCDIT12");
        variables.put("paymentTerms", 30);

        emailService.sendEmailWithAttachment(
            invoice.getCustomer().getEmail(),
            "Invoice " + invoice.getNumber(),
            "invoice",
            variables,
            "invoice-" + invoice.getNumber() + ".pdf",
            pdfData
        );
    }
}
```

## Testing Email Templates

```java
@SpringBootTest
@AutoConfigureMockMvc
class EmailServiceTest {

    @Autowired
    private EmailService emailService;

    @MockBean
    private JavaMailSender mailSender;

    @Test
    @DisplayName("Should send welcome email successfully")
    void shouldSendWelcomeEmail() throws MessagingException {
        // Given
        MimeMessage mimeMessage = mock(MimeMessage.class);
        when(mailSender.createMimeMessage()).thenReturn(mimeMessage);

        Map<String, Object> variables = new HashMap<>();
        variables.put("userName", "John Doe");
        variables.put("userEmail", "john@example.com");
        variables.put("registrationDate", LocalDateTime.now());
        variables.put("verificationUrl", "http://localhost/verify?token=abc123");
        variables.put("expirationHours", 24);
        variables.put("baseUrl", "http://localhost");

        // When
        emailService.sendEmail(
            "john@example.com",
            "Welcome",
            "user-registration",
            variables
        );

        // Then
        verify(mailSender).send(any(MimeMessage.class));
    }
}
```

## Best Practices

1. ✅ Use inline CSS for maximum email client compatibility
2. ✅ Test emails in multiple email clients (Gmail, Outlook, Apple Mail)
3. ✅ Keep email width at 600px maximum
4. ✅ Use tables for layout (better email client support)
5. ✅ Always provide plain text alternatives
6. ✅ Include unsubscribe links for marketing emails
7. ✅ Use absolute URLs for images and links
8. ✅ Optimize images and use alt text
9. ✅ Handle email sending asynchronously for better performance
10. ✅ Log email sending for debugging and auditing

## References

- [Thymeleaf Documentation](https://www.thymeleaf.org/documentation.html)
- [Spring Boot Mail](https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.email)
- [Email Client CSS Support](https://www.campaignmonitor.com/css/)
- [HTML Email Best Practices](https://www.emailonacid.com/blog/)
