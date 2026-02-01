# Text Styling Migration Guide

This guide explains how to migrate from legacy § color codes to the modern Messages API.

## Why Migrate?

The legacy § color code system has several issues:

1. **Platform-dependent**: § codes only work with the Hytale adapter's parsing layer
2. **Hard to read**: `"§cError: §7" + message` is less readable than semantic alternatives
3. **No IDE support**: No autocomplete or validation for color codes
4. **Limited styling**: No hover events, click actions, or gradients

## The Messages API

The `Messages` utility class in `03-framework-text-styling` provides:

- **Semantic message types**: `error()`, `success()`, `info()`, `warning()`
- **Structured messages**: `header()`, `labeled()`, `bullet()`, `commandHint()`
- **Fluent builder**: Chain styling methods for complex messages
- **Legacy bridge**: Convert between Components and § strings

## Quick Reference

### Before (Legacy § Codes)

```java
sender.sendMessage("§cError: Something went wrong!");
sender.sendMessage("§aSuccess! Quest completed.");
sender.sendMessage("§6=== Quest Log ===");
sender.sendMessage("§7Name: §fSteve");
sender.sendMessage("§e/help §7- Show help message");
```

### After (Messages API)

```java
import com.argonathsystems.framework.text.Messages;

// Using legacy() bridge for compatibility with string-based APIs
sender.sendMessage(Messages.legacy(Messages.error("Something went wrong!")));
sender.sendMessage(Messages.legacy(Messages.success("Quest completed.")));
sender.sendMessage(Messages.legacy(Messages.header("Quest Log")));
sender.sendMessage(Messages.legacy(Messages.labeled("Name", "Steve")));
sender.sendMessage(Messages.legacy(Messages.commandHint("help", "Show help message")));
```

### With Static Import (Cleaner)

```java
import static com.argonathsystems.framework.text.Messages.*;

sender.sendMessage(legacy(error("Something went wrong!")));
sender.sendMessage(legacy(success("Quest completed.")));
sender.sendMessage(legacy(header("Quest Log")));
sender.sendMessage(legacy(labeled("Name", "Steve")));
sender.sendMessage(legacy(commandHint("help", "Show help message")));
```

## Message Types

### Quick Messages

| Method | Color | Use Case |
|--------|-------|----------|
| `error(msg)` | Red | Error messages, failures |
| `success(msg)` | Green | Success confirmations |
| `info(msg)` | Gray | Informational messages |
| `warning(msg)` | Yellow | Warnings, cautions |
| `highlight(msg)` | Gold | Important highlights |
| `muted(msg)` | Dark Gray | Secondary information |

### Structured Messages

| Method | Example Output | Use Case |
|--------|----------------|----------|
| `header("Title")` | `=== Title ===` (gold) | Section headers |
| `subheader("Title")` | `- Title -` (yellow) | Subsection headers |
| `labeled("Key", "Value")` | `Key: Value` (gray/white) | Key-value display |
| `bullet("Item")` | `• Item` | List items |
| `commandHint("/cmd", "desc")` | `/cmd - desc` | Command help |
| `status("Active", true)` | `Active: ✓ Yes` | Boolean status |
| `progress(3, 10)` | `(3/10)` (yellow) | Progress display |

## Builder for Complex Messages

For complex multi-styled messages, use the fluent builder:

```java
Component message = Messages.builder()
    .gold().bold().text("=== ")
    .gold().text("Quest Log")
    .gold().bold().text(" ===")
    .newline()
    .gray().text("Active quests: ")
    .green().text("3")
    .build();

sender.sendMessage(Messages.legacy(message));
```

### Builder Methods

**Colors**: `black()`, `darkBlue()`, `darkGreen()`, `darkAqua()`, `darkRed()`, 
`darkPurple()`, `gold()`, `gray()`, `darkGray()`, `blue()`, `green()`, `aqua()`, 
`red()`, `lightPurple()`, `yellow()`, `white()`, `color(TextColor)`

**Decorations**: `bold()`, `italic()`, `underlined()`, `strikethrough()`

**Control**: `reset()`, `text(String)`, `space()`, `newline()`, `append(Component)`

## Legacy Compatibility

### Convert § String to Component

```java
String legacyString = "§cError: §7Something went wrong";
Component component = Messages.fromLegacy(legacyString);
```

### Convert Component to § String

```java
Component errorMsg = Messages.error("Something went wrong");
String legacyString = Messages.toLegacy(errorMsg);
// Result: "§cSomething went wrong"
```

## MiniMessage Format

For XML-like formatting, use MiniMessage:

```java
Component msg = Messages.parse("<red>Error: </red><gray>Something went wrong</gray>");
Component fancy = Messages.parse("<gradient:red:gold>Legendary Item!</gradient>");
```

### Supported Tags

- **Colors**: `<red>`, `<blue>`, `<#FF5500>`
- **Decorations**: `<bold>`, `<italic>`, `<underlined>`, `<strikethrough>`
- **Gradients**: `<gradient:red:gold>text</gradient>`
- **Hover**: `<hover:show_text:Tooltip>text</hover>`
- **Click**: `<click:run_command:/help>text</click>`

## Migration Strategy

### Phase 1: Use Legacy Bridge (Now)

Wrap new code with `Messages.legacy()`:

```java
// Instead of:
sender.sendMessage("§cUnknown command.");

// Use:
sender.sendMessage(Messages.legacy(Messages.error("Unknown command.")));
```

### Phase 2: Component-Aware APIs (Future)

When CommandSender supports Components:

```java
sender.sendComponent(Messages.error("Unknown command."));
```

## Best Practices

1. **Use semantic methods** - `error()` instead of `"§c"`
2. **Static import** - `import static ...Messages.*` for cleaner code
3. **Prefer builder for complex messages** - More readable than concatenation
4. **Avoid mixing styles** - Don't mix § codes and Messages API in same message
5. **Test with different clients** - Ensure colors display correctly

## Color Code Reference

For reference, here are the § codes and their `TextColor` equivalents:

| § Code | TextColor Constant | Hex |
|--------|-------------------|-----|
| §0 | `TextColor.BLACK` | #000000 |
| §1 | `TextColor.DARK_BLUE` | #0000AA |
| §2 | `TextColor.DARK_GREEN` | #00AA00 |
| §3 | `TextColor.DARK_AQUA` | #00AAAA |
| §4 | `TextColor.DARK_RED` | #AA0000 |
| §5 | `TextColor.DARK_PURPLE` | #AA00AA |
| §6 | `TextColor.GOLD` | #FFAA00 |
| §7 | `TextColor.GRAY` | #AAAAAA |
| §8 | `TextColor.DARK_GRAY` | #555555 |
| §9 | `TextColor.BLUE` | #5555FF |
| §a | `TextColor.GREEN` | #55FF55 |
| §b | `TextColor.AQUA` | #55FFFF |
| §c | `TextColor.RED` | #FF5555 |
| §d | `TextColor.LIGHT_PURPLE` | #FF55FF |
| §e | `TextColor.YELLOW` | #FFFF55 |
| §f | `TextColor.WHITE` | #FFFFFF |

## Related Documentation

- [Text Styling Framework API](/api-reference/text-styling/)
- [UI Development Guide](ui-development.md)
- [Command Development Best Practices](/guides/commands/)
