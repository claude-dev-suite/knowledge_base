# Spring Shell - Commands

## Overview

Spring Shell provides annotations and patterns for creating interactive command-line commands with parameters, validation, and help text.

## Basic Commands

### Simple Command
```java
@ShellComponent
public class BasicCommands {

    @ShellMethod("Display a greeting message")
    public String greet() {
        return "Hello, World!";
    }

    @ShellMethod(value = "Say hello to someone", key = "say-hello")
    public String sayHello(String name) {
        return "Hello, " + name + "!";
    }

    @ShellMethod(key = {"quit", "exit", "bye"}, value = "Exit the shell")
    public void quit() {
        System.exit(0);
    }
}
```

### Command with Parameters
```java
@ShellComponent
public class UserCommands {

    @ShellMethod("Create a new user")
    public String createUser(
            @ShellOption(help = "The user's name") String name,
            @ShellOption(help = "The user's email") String email,
            @ShellOption(defaultValue = "USER", help = "The user's role") String role) {

        return String.format("Created user: %s <%s> [%s]", name, email, role);
    }

    @ShellMethod("Find users by criteria")
    public List<String> findUsers(
            @ShellOption(value = {"-n", "--name"}, defaultValue = ShellOption.NULL) String name,
            @ShellOption(value = {"-e", "--email"}, defaultValue = ShellOption.NULL) String email,
            @ShellOption(value = {"-l", "--limit"}, defaultValue = "10") int limit) {

        // Search logic
        return userService.search(name, email, limit);
    }
}
```

### Boolean and Flag Options
```java
@ShellComponent
public class OptionsCommands {

    @ShellMethod("List files")
    public String listFiles(
            @ShellOption(value = {"-a", "--all"}, defaultValue = "false") boolean showAll,
            @ShellOption(value = {"-l", "--long"}, defaultValue = "false") boolean longFormat,
            @ShellOption(value = {"-h", "--human"}, defaultValue = "false") boolean humanReadable) {

        StringBuilder result = new StringBuilder("Files:\n");
        // Build output based on flags
        return result.toString();
    }

    @ShellMethod("Process data")
    public String process(
            @ShellOption(arity = 0, value = "--verbose") boolean verbose,
            @ShellOption(arity = 0, value = "--dry-run") boolean dryRun) {

        if (dryRun) {
            return "Dry run - no changes made";
        }
        return verbose ? "Detailed output..." : "Done";
    }
}
```

### Array Parameters
```java
@ShellComponent
public class ArrayCommands {

    @ShellMethod("Add tags to an item")
    public String addTags(
            @ShellOption String itemId,
            @ShellOption(arity = 3) String[] tags) {

        return String.format("Added tags %s to item %s",
            Arrays.toString(tags), itemId);
    }

    @ShellMethod("Process multiple files")
    public String processFiles(
            @ShellOption(value = "--files", arity = -1) String[] files) {

        if (files == null || files.length == 0) {
            return "No files specified";
        }
        return String.format("Processing %d files", files.length);
    }
}
```

## Command Groups

```java
@ShellComponent
@ShellCommandGroup("User Management")
public class UserManagementCommands {

    @ShellMethod("List all users")
    public List<User> listUsers() { ... }

    @ShellMethod("Add a new user")
    public String addUser(String name) { ... }

    @ShellMethod("Remove a user")
    public String removeUser(String id) { ... }
}

@ShellComponent
@ShellCommandGroup("System")
public class SystemCommands {

    @ShellMethod("Show system status")
    public String status() { ... }

    @ShellMethod("Clear the screen")
    public void clear() { ... }
}
```

## Input Validation

```java
@ShellComponent
public class ValidatedCommands {

    @ShellMethod("Add a user with validation")
    public String addUser(
            @ShellOption @NotBlank(message = "Name is required") String name,
            @ShellOption @Email(message = "Invalid email format") String email,
            @ShellOption @Min(value = 18, message = "Age must be at least 18") int age) {

        return String.format("Added: %s (%d) - %s", name, age, email);
    }

    @ShellMethod("Transfer funds")
    public String transfer(
            @ShellOption @NotBlank String fromAccount,
            @ShellOption @NotBlank String toAccount,
            @ShellOption @Positive(message = "Amount must be positive") BigDecimal amount) {

        return String.format("Transferred %s from %s to %s",
            amount, fromAccount, toAccount);
    }
}
```

## Command Availability

```java
@ShellComponent
public class ConditionalCommands {

    private boolean authenticated = false;
    private String currentUser;

    @ShellMethod("Login to the system")
    public String login(String username, String password) {
        if (authService.authenticate(username, password)) {
            authenticated = true;
            currentUser = username;
            return "Login successful";
        }
        return "Login failed";
    }

    @ShellMethod("View protected data")
    public String viewData() {
        return "Sensitive data for " + currentUser;
    }

    @ShellMethodAvailability("viewData")
    public Availability viewDataAvailability() {
        return authenticated
            ? Availability.available()
            : Availability.unavailable("You must login first. Use 'login <user> <pass>'");
    }

    @ShellMethod("Logout")
    public String logout() {
        authenticated = false;
        currentUser = null;
        return "Logged out";
    }

    @ShellMethodAvailability("logout")
    public Availability logoutAvailability() {
        return authenticated
            ? Availability.available()
            : Availability.unavailable("Not logged in");
    }

    // Apply to multiple commands
    @ShellMethodAvailability({"viewData", "logout", "editProfile"})
    public Availability requiresAuth() {
        return authenticated
            ? Availability.available()
            : Availability.unavailable("Authentication required");
    }
}
```

## Return Types

```java
@ShellComponent
public class ReturnTypeCommands {

    // String - displayed as-is
    @ShellMethod("Return string")
    public String stringResult() {
        return "Simple string result";
    }

    // List/Collection - each item on new line
    @ShellMethod("Return list")
    public List<String> listResult() {
        return List.of("Item 1", "Item 2", "Item 3");
    }

    // Table - formatted table output
    @ShellMethod("Return table")
    public Table tableResult() {
        TableModel model = new BeanListTableModel<>(
            userService.findAll(),
            new String[]{"id", "name", "email"}
        );

        return new TableBuilder(model)
            .addHeaderRow()
            .addOutlineBorder(BorderStyle.fancy_light)
            .build();
    }

    // Void - no output
    @ShellMethod("Side effect only")
    public void sideEffect() {
        // Do something, no output
    }

    // AttributedString - styled output
    @ShellMethod("Styled output")
    public AttributedString styledResult() {
        return new AttributedString(
            "Success!",
            AttributedStyle.DEFAULT.foreground(AttributedStyle.GREEN)
        );
    }
}
```

## Exception Handling

```java
@ShellComponent
public class ErrorHandlingCommands {

    @ShellMethod("Command that may fail")
    public String riskyCommand(String param) {
        if ("fail".equals(param)) {
            throw new IllegalArgumentException("Invalid parameter: " + param);
        }
        return "Success";
    }
}

@Component
public class CustomExceptionResolver implements CommandExceptionResolver {

    @Override
    public boolean handles(Throwable t) {
        return t instanceof CustomException;
    }

    @Override
    public void resolve(Throwable t, Terminal terminal) {
        terminal.writer().println(
            new AttributedString(
                "Error: " + t.getMessage(),
                AttributedStyle.DEFAULT.foreground(AttributedStyle.RED)
            ).toAnsi()
        );
    }
}
```

## Dynamic Command Registration

```java
@Component
public class DynamicCommandRegistrar {

    private final CommandRegistry commandRegistry;
    private final ApplicationContext context;

    public void registerCommand(String name, Runnable action) {
        // Dynamic command registration
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use clear command names | Cryptic abbreviations |
| Provide help text | Skip documentation |
| Validate input | Trust all input |
| Use @ShellOption for flags | Complex parameter parsing |
| Implement availability checks | Show unavailable commands |
| Group related commands | Flat command structure |

## Production Checklist

- [ ] Commands documented with help text
- [ ] Input validation in place
- [ ] Availability checks for conditional commands
- [ ] Exception handling configured
- [ ] Command groups organized
- [ ] Output formatting consistent
