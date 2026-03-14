# Spring Shell - Components

## Overview

Spring Shell provides interactive UI components including progress indicators, tables, selectors, and input components for building rich CLI applications.

## Progress Indicators

### Progress View
```java
@ShellComponent
public class ProgressCommands {

    private final Terminal terminal;

    @ShellMethod("Process with progress bar")
    public void processWithProgress() {
        List<String> items = getItemsToProcess();

        ProgressView progress = new ProgressView();
        progress.setTerminal(terminal);
        progress.start();

        for (int i = 0; i < items.size(); i++) {
            processItem(items.get(i));
            int percent = ((i + 1) * 100) / items.size();
            progress.display(percent);
        }

        progress.stop();
    }
}
```

### Spinner
```java
@ShellComponent
public class SpinnerCommands {

    private final Terminal terminal;

    @ShellMethod("Long running task with spinner")
    public String longTask() throws InterruptedException {
        Spinner spinner = new Spinner(terminal, "Processing...");
        spinner.start();

        try {
            // Simulate work
            Thread.sleep(3000);
            return "Done!";
        } finally {
            spinner.stop();
        }
    }
}
```

## Tables

### Simple Table
```java
@ShellComponent
public class TableCommands {

    @ShellMethod("Display users as table")
    public Table listUsers() {
        String[][] data = {
            {"1", "John Doe", "john@example.com"},
            {"2", "Jane Smith", "jane@example.com"},
            {"3", "Bob Wilson", "bob@example.com"}
        };

        TableModel model = new ArrayTableModel(data);

        return new TableBuilder(model)
            .addHeaderRow()
            .addInnerBorder(BorderStyle.fancy_light)
            .addOutlineBorder(BorderStyle.fancy_double)
            .build();
    }

    @ShellMethod("Display data with headers")
    public Table listWithHeaders() {
        TableModel model = new ArrayTableModel(new String[][]{
            {"ID", "Name", "Status"},
            {"1", "Service A", "Running"},
            {"2", "Service B", "Stopped"},
            {"3", "Service C", "Running"}
        });

        return new TableBuilder(model)
            .addHeaderRow()
            .addFullBorder(BorderStyle.fancy_light)
            .build();
    }
}
```

### Bean Table
```java
@ShellComponent
public class BeanTableCommands {

    private final UserService userService;

    @ShellMethod("Display users from service")
    public Table listUsersFromService() {
        List<User> users = userService.findAll();

        TableModel model = new BeanListTableModel<>(
            users,
            new String[]{"id", "username", "email", "role"}
        );

        TableBuilder builder = new TableBuilder(model);
        builder.addHeaderRow();
        builder.addFullBorder(BorderStyle.fancy_light);

        // Custom column alignment
        builder.on(CellMatchers.column(0)).addAligner(SimpleHorizontalAligner.right);
        builder.on(CellMatchers.column(3)).addAligner(SimpleHorizontalAligner.center);

        return builder.build();
    }
}
```

### Styled Table
```java
@ShellMethod("Display styled table")
public Table styledTable() {
    String[][] data = {
        {"Service", "Status", "Uptime"},
        {"API", "UP", "99.9%"},
        {"Database", "UP", "99.8%"},
        {"Cache", "DOWN", "0%"}
    };

    TableModel model = new ArrayTableModel(data);
    TableBuilder builder = new TableBuilder(model);

    builder.addHeaderRow();
    builder.addFullBorder(BorderStyle.fancy_double);

    // Style cells based on content
    builder.on(CellMatchers.ofType(String.class))
        .addSizer((text, width, height) -> {
            if ("UP".equals(text)) {
                return new AttributedString(text,
                    AttributedStyle.DEFAULT.foreground(AttributedStyle.GREEN));
            } else if ("DOWN".equals(text)) {
                return new AttributedString(text,
                    AttributedStyle.DEFAULT.foreground(AttributedStyle.RED));
            }
            return new AttributedString(text);
        });

    return builder.build();
}
```

## Selection Components

### Single Item Selector
```java
@ShellComponent
public class SelectorCommands {

    private final Terminal terminal;

    @ShellMethod("Select an environment")
    public String selectEnvironment() {
        List<SelectorItem<String>> items = List.of(
            SelectorItem.of("Development", "dev"),
            SelectorItem.of("Staging", "staging"),
            SelectorItem.of("Production", "prod")
        );

        SingleItemSelector<String, SelectorItem<String>> selector =
            new SingleItemSelector<>(terminal, items, "Select environment", null);

        selector.setDefaultSelect(items.get(0));

        SingleItemSelector.SingleItemSelectorContext<String, SelectorItem<String>> context =
            selector.run(SingleItemSelector.SingleItemSelectorContext.empty());

        return context.getResultItem()
            .map(item -> "Selected: " + item.getItem())
            .orElse("No selection");
    }
}
```

### Multi-Item Selector
```java
@ShellMethod("Select features to enable")
public String selectFeatures() {
    List<SelectorItem<String>> items = List.of(
        SelectorItem.of("Logging", "logging"),
        SelectorItem.of("Metrics", "metrics"),
        SelectorItem.of("Tracing", "tracing"),
        SelectorItem.of("Security", "security")
    );

    MultiItemSelector<String, SelectorItem<String>> selector =
        new MultiItemSelector<>(terminal, items, "Select features", null);

    // Pre-select some items
    selector.setDefaultSelect(Set.of(items.get(0), items.get(1)));

    MultiItemSelector.MultiItemSelectorContext<String, SelectorItem<String>> context =
        selector.run(MultiItemSelector.MultiItemSelectorContext.empty());

    List<String> selected = context.getResultItems().stream()
        .map(SelectorItem::getItem)
        .collect(Collectors.toList());

    return "Selected features: " + String.join(", ", selected);
}
```

## Input Components

### Text Input
```java
@ShellComponent
public class InputCommands {

    private final Terminal terminal;

    @ShellMethod("Get user input")
    public String getUserInput() {
        StringInput input = new StringInput(terminal, "Enter your name", null);
        input.setDefaultValue("Anonymous");

        StringInput.StringInputContext context =
            input.run(StringInput.StringInputContext.empty());

        return "Hello, " + context.getResultValue() + "!";
    }

    @ShellMethod("Get password input")
    public String getPassword() {
        StringInput input = new StringInput(terminal, "Enter password", null);
        input.setMaskCharacter('*');

        StringInput.StringInputContext context =
            input.run(StringInput.StringInputContext.empty());

        return "Password received (length: " + context.getResultValue().length() + ")";
    }
}
```

### Path Input
```java
@ShellMethod("Select a file")
public String selectFile() {
    PathInput input = new PathInput(terminal, "Select file", null);
    input.setRoot(Paths.get("."));

    PathInput.PathInputContext context =
        input.run(PathInput.PathInputContext.empty());

    return "Selected: " + context.getResultValue();
}
```

### Confirmation Input
```java
@ShellMethod("Confirm action")
public String confirmAction() {
    ConfirmationInput input = new ConfirmationInput(
        terminal,
        "Are you sure you want to proceed?",
        null
    );

    ConfirmationInput.ConfirmationInputContext context =
        input.run(ConfirmationInput.ConfirmationInputContext.empty());

    if (context.getResultValue()) {
        return "Action confirmed";
    } else {
        return "Action cancelled";
    }
}
```

## Component Flow

```java
@ShellComponent
public class FlowCommands {

    private final ComponentFlow.Builder flowBuilder;

    @ShellMethod("Create user with wizard")
    public String createUserWizard() {
        ComponentFlow flow = flowBuilder.clone()
            .reset()
            .withStringInput("name")
                .name("Username")
                .defaultValue("")
                .and()
            .withStringInput("email")
                .name("Email")
                .defaultValue("")
                .and()
            .withSingleItemSelector("role")
                .name("Role")
                .selectItems(List.of(
                    SelectItem.of("Admin", "ADMIN"),
                    SelectItem.of("User", "USER"),
                    SelectItem.of("Guest", "GUEST")
                ))
                .and()
            .withConfirmationInput("confirm")
                .name("Create user?")
                .and()
            .build();

        ComponentFlowResult result = flow.run();

        if (result.getContext().get("confirm", Boolean.class)) {
            String name = result.getContext().get("name", String.class);
            String email = result.getContext().get("email", String.class);
            String role = result.getContext().get("role", String.class);

            return String.format("Created user: %s <%s> [%s]", name, email, role);
        }

        return "User creation cancelled";
    }
}
```

## Custom Components

```java
public class CustomProgressComponent {

    private final Terminal terminal;
    private final int total;
    private int current;

    public void update(int progress, String message) {
        terminal.writer().print("\r");

        int width = 40;
        int filled = (progress * width) / 100;

        StringBuilder bar = new StringBuilder("[");
        bar.append("=".repeat(filled));
        bar.append(" ".repeat(width - filled));
        bar.append("] ");
        bar.append(progress).append("% ");
        bar.append(message);

        terminal.writer().print(bar);
        terminal.writer().flush();
    }

    public void complete() {
        terminal.writer().println();
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use component flows for wizards | Complex manual input handling |
| Provide sensible defaults | Require all input every time |
| Style tables appropriately | Unstyled raw output |
| Show progress for long tasks | Leave user wondering |
| Handle cancellation gracefully | Force completion |
| Use confirmation for destructive ops | Skip confirmation |

## Production Checklist

- [ ] Progress indicators for long tasks
- [ ] Tables formatted properly
- [ ] Input validation on components
- [ ] Cancellation handling
- [ ] Keyboard navigation works
- [ ] Color schemes accessible
