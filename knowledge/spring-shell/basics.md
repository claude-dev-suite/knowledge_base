# Spring Shell - Basics

## Overview

Spring Shell provides an interactive command-line interface framework for building CLI applications with Spring Boot.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.shell</groupId>
    <artifactId>spring-shell-starter</artifactId>
</dependency>
```

## Basic Commands

### Simple Command
```java
@ShellComponent
public class GreetingCommands {

    @ShellMethod("Say hello")
    public String hello() {
        return "Hello, World!";
    }

    @ShellMethod("Greet a person by name")
    public String greet(@ShellOption String name) {
        return "Hello, " + name + "!";
    }

    @ShellMethod(key = "say-goodbye", value = "Say goodbye to someone")
    public String goodbye(
            @ShellOption(defaultValue = "World") String name) {
        return "Goodbye, " + name + "!";
    }
}
```

### Command Options
```java
@ShellComponent
public class UserCommands {

    @ShellMethod("Create a new user")
    public String createUser(
            @ShellOption(help = "User's name") String name,
            @ShellOption(help = "User's email") String email,
            @ShellOption(defaultValue = "USER", help = "User's role") String role,
            @ShellOption(value = {"-a", "--admin"}, defaultValue = "false") boolean isAdmin) {

        // Create user
        return String.format("Created user: %s <%s> [%s]%s",
            name, email, role, isAdmin ? " (admin)" : "");
    }
}
```

## Input Validation

```java
@ShellComponent
public class ValidatedCommands {

    @ShellMethod("Add a user with validated input")
    public String addUser(
            @ShellOption @NotBlank String name,
            @ShellOption @Email String email,
            @ShellOption @Min(18) @Max(120) int age) {

        return String.format("Added user: %s (%d) - %s", name, age, email);
    }
}
```

## Command Groups

```java
@ShellComponent
@ShellCommandGroup("User Management")
public class UserManagementCommands {

    @ShellMethod("List all users")
    public String listUsers() {
        return userService.findAll().toString();
    }

    @ShellMethod("Show user details")
    public String showUser(@ShellOption String id) {
        return userService.findById(id).toString();
    }
}

@ShellComponent
@ShellCommandGroup("System")
public class SystemCommands {

    @ShellMethod("Show system status")
    public String status() {
        return "System is running";
    }
}
```

## Availability

Control when commands are available:

```java
@ShellComponent
public class SecureCommands {

    private boolean authenticated = false;

    @ShellMethod("Login to the system")
    public String login(String username, String password) {
        if (authService.authenticate(username, password)) {
            authenticated = true;
            return "Login successful";
        }
        return "Login failed";
    }

    @ShellMethod("View sensitive data")
    public String viewData() {
        return "Sensitive data: ...";
    }

    @ShellMethodAvailability("viewData")
    public Availability viewDataAvailability() {
        return authenticated
            ? Availability.available()
            : Availability.unavailable("You must login first");
    }

    @ShellMethod("Logout")
    public String logout() {
        authenticated = false;
        return "Logged out";
    }
}
```

## Interactive Components

### Progress Bar
```java
@ShellComponent
public class ProgressCommands {

    private final TerminalUI ui;

    @ShellMethod("Process files with progress")
    public void processFiles() {
        List<String> files = getFiles();

        try (ProgressView progress = new ProgressView()) {
            progress.start();
            for (int i = 0; i < files.size(); i++) {
                processFile(files.get(i));
                progress.display((i + 1) * 100 / files.size());
            }
        }
    }
}
```

### Tables
```java
@ShellComponent
public class TableCommands {

    @ShellMethod("List users in a table")
    public Table listUsers() {
        List<User> users = userService.findAll();

        String[][] data = users.stream()
            .map(u -> new String[]{
                u.getId().toString(),
                u.getName(),
                u.getEmail()
            })
            .toArray(String[][]::new);

        return new TableBuilder(new ArrayTableModel(data))
            .addHeaderRow("ID", "Name", "Email")
            .build();
    }
}
```

### Selection
```java
@ShellComponent
public class SelectionCommands {

    private final Terminal terminal;
    private final ComponentFlow flow;

    @ShellMethod("Select an option")
    public String selectOption() {
        List<SelectorItem<String>> items = Arrays.asList(
            SelectorItem.of("Option 1", "value1"),
            SelectorItem.of("Option 2", "value2"),
            SelectorItem.of("Option 3", "value3")
        );

        SingleItemSelector<String, SelectorItem<String>> selector =
            new SingleItemSelector<>(terminal, items, "Select an option", null);

        selector.setDefaultSelect(items.get(0));
        SingleItemSelector.SingleItemSelectorContext<String, SelectorItem<String>> context =
            selector.run(SingleItemSelector.SingleItemSelectorContext.empty());

        return "Selected: " + context.getResultItem().map(SelectorItem::getItem).orElse("none");
    }
}
```

## Non-Interactive Mode

```bash
# Run single command
java -jar app.jar greet --name John

# Run script
java -jar app.jar @commands.txt

# Interactive mode
java -jar app.jar
```

## Configuration

### application.yml
```yaml
spring:
  shell:
    interactive:
      enabled: true
    script:
      enabled: true
    command:
      help:
        enabled: true
      clear:
        enabled: true
      quit:
        enabled: true
      history:
        enabled: true
```

## Custom Prompt

```java
@Component
public class CustomPromptProvider implements PromptProvider {

    private String currentDirectory = "/";

    @Override
    public AttributedString getPrompt() {
        return new AttributedString(
            currentDirectory + " $ ",
            AttributedStyle.DEFAULT.foreground(AttributedStyle.GREEN)
        );
    }

    public void setCurrentDirectory(String dir) {
        this.currentDirectory = dir;
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Group related commands | Flat command structure |
| Provide helpful descriptions | Skip documentation |
| Validate input | Trust user input |
| Use availability checks | Show unavailable commands |
| Handle errors gracefully | Let exceptions propagate |
| Support both interactive and script modes | Interactive only |

## Production Checklist

- [ ] Commands documented
- [ ] Input validation in place
- [ ] Proper error handling
- [ ] Availability checks for sensitive commands
- [ ] Help text for all options
- [ ] Non-interactive mode tested
