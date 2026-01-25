# UI Development Guide

A comprehensive guide to developing user interfaces with the Argonath Framework UI System.

## Table of Contents

1. [Introduction](#introduction)
2. [UI Framework Overview](#ui-framework-overview)
3. [Creating Windows](#creating-windows)
4. [UI Components](#ui-components)
5. [Layout Systems](#layout-systems)
6. [Event Handling](#event-handling)
7. [Styling and Theming](#styling-and-theming)
8. [Quest Tracker Example](#quest-tracker-example)
9. [Custom Components](#custom-components)
10. [Best Practices](#best-practices)
11. [Performance Optimization](#performance-optimization)
12. [Troubleshooting](#troubleshooting)

---

## Introduction

The Argonath UI Framework provides a powerful, flexible system for creating custom user interfaces in Hytale. Build everything from simple dialogs to complex inventory systems and quest trackers.

### Prerequisites

- Understanding of [Core Concepts](../architecture/concepts.md)
- Basic Java knowledge
- Familiarity with event-driven programming

### What You'll Learn

- UI framework architecture
- Creating windows and dialogs
- Working with UI components
- Layout management
- Event handling patterns
- Styling and theming
- Performance optimization

---

## UI Framework Overview

### Architecture

```
UIManager
    └── UIWindow (Container)
        └── UIPanel (Layout Container)
            ├── UIButton
            ├── UILabel
            ├── UITextField
            ├── UIList
            └── Custom Components
```

### Core Components

```java
public class UIFramework {
    // Component types
    public interface UIComponent {
        void render(RenderContext ctx);
        void update(float deltaTime);
        void handleEvent(UIEvent event);
        Rectangle getBounds();
    }
    
    // Container types
    public interface UIContainer extends UIComponent {
        void addChild(UIComponent component);
        void removeChild(UIComponent component);
        List<UIComponent> getChildren();
    }
    
    // Window types
    public enum WindowType {
        NORMAL,      // Standard window
        MODAL,       // Blocks interaction with other windows
        POPUP,       // Small notification
        OVERLAY,     // Full-screen overlay
        HUD          // Always visible HUD element
    }
}
```

### Basic Window Creation

```java
import com.argonath.framework.ui.*;

public class BasicUI {
    public static UIWindow createSimpleWindow() {
        return new UIWindowBuilder("simple_window")
            .withTitle("My Window")
            .withSize(400, 300)
            .withPosition(100, 100)
            .closeable(true)
            .draggable(true)
            .build();
    }
}
```

---

## Creating Windows

### Window Builder Pattern

```java
public class WindowCreation {
    public static UIWindow createQuestWindow() {
        return new UIWindowBuilder("quest_window")
            .withTitle("Quests")
            .withSize(600, 400)
            .centered()
            .withType(WindowType.NORMAL)
            .closeable(true)
            .draggable(true)
            .resizable(true)
            .withMinSize(400, 300)
            .withMaxSize(800, 600)
            .withBackgroundColor(0x222222)
            .withBorderColor(0x444444)
            .withTitleBarColor(0x333333)
            .onOpen(window -> {
                System.out.println("Quest window opened");
            })
            .onClose(window -> {
                System.out.println("Quest window closed");
            })
            .build();
    }
    
    public static UIWindow createModalDialog() {
        return new UIWindowBuilder("confirm_dialog")
            .withTitle("Confirm Action")
            .withSize(300, 150)
            .centered()
            .withType(WindowType.MODAL)
            .closeable(false)
            .draggable(false)
            .withBackdrop(true)
            .withBackdropOpacity(0.5f)
            .build();
    }
    
    public static UIWindow createHUD() {
        return new UIWindowBuilder("quest_hud")
            .withTitle("")
            .withSize(250, 200)
            .atPosition(10, 10)
            .withType(WindowType.HUD)
            .closeable(false)
            .draggable(false)
            .withBackground(false)
            .withBorder(false)
            .build();
    }
}
```

### Window Management

```java
public class WindowManager {
    private final Map<String, UIWindow> windows = new HashMap<>();
    private final UIManager uiManager;
    
    public void openWindow(String id) {
        UIWindow window = windows.get(id);
        if (window != null) {
            uiManager.showWindow(window);
        }
    }
    
    public void closeWindow(String id) {
        UIWindow window = windows.get(id);
        if (window != null) {
            uiManager.hideWindow(window);
        }
    }
    
    public void toggleWindow(String id) {
        UIWindow window = windows.get(id);
        if (window != null) {
            if (window.isVisible()) {
                uiManager.hideWindow(window);
            } else {
                uiManager.showWindow(window);
            }
        }
    }
    
    public boolean isWindowOpen(String id) {
        UIWindow window = windows.get(id);
        return window != null && window.isVisible();
    }
}
```

---

## UI Components

### Labels

```java
public class LabelExamples {
    public static UILabel createSimpleLabel() {
        return new UILabelBuilder()
            .withText("Hello, World!")
            .withPosition(10, 10)
            .withSize(200, 20)
            .withFont("default")
            .withFontSize(14)
            .withColor(0xFFFFFF)
            .withAlignment(TextAlignment.LEFT)
            .build();
    }
    
    public static UILabel createTitleLabel() {
        return new UILabelBuilder()
            .withText("Quest Log")
            .withPosition(0, 0)
            .withSize(400, 40)
            .withFont("title")
            .withFontSize(24)
            .withColor(0xFFD700)
            .withAlignment(TextAlignment.CENTER)
            .bold(true)
            .build();
    }
    
    public static UILabel createDynamicLabel(Supplier<String> textProvider) {
        UILabel label = new UILabelBuilder()
            .withText("")
            .withPosition(10, 10)
            .withSize(200, 20)
            .build();
        
        label.setTextProvider(textProvider);
        return label;
    }
}
```

### Buttons

```java
public class ButtonExamples {
    public static UIButton createSimpleButton() {
        return new UIButtonBuilder()
            .withText("Click Me")
            .withPosition(10, 10)
            .withSize(100, 30)
            .withBackgroundColor(0x4CAF50)
            .withHoverColor(0x45A049)
            .withPressedColor(0x3D8B40)
            .onClick(event -> {
                System.out.println("Button clicked!");
            })
            .build();
    }
    
    public static UIButton createIconButton(String iconPath) {
        return new UIButtonBuilder()
            .withIcon(iconPath)
            .withPosition(10, 10)
            .withSize(32, 32)
            .withTooltip("Icon Button")
            .onClick(event -> {
                System.out.println("Icon button clicked!");
            })
            .build();
    }
    
    public static UIButton createToggleButton() {
        UIButton button = new UIButtonBuilder()
            .withText("Toggle")
            .withPosition(10, 10)
            .withSize(100, 30)
            .toggleable(true)
            .build();
        
        button.onToggle(toggled -> {
            if (toggled) {
                button.setText("ON");
                button.setBackgroundColor(0x4CAF50);
            } else {
                button.setText("OFF");
                button.setBackgroundColor(0x888888);
            }
        });
        
        return button;
    }
}
```

### Text Fields

```java
public class TextFieldExamples {
    public static UITextField createSimpleTextField() {
        return new UITextFieldBuilder()
            .withPlaceholder("Enter text...")
            .withPosition(10, 10)
            .withSize(200, 30)
            .withMaxLength(50)
            .onChange(text -> {
                System.out.println("Text changed: " + text);
            })
            .build();
    }
    
    public static UITextField createPasswordField() {
        return new UITextFieldBuilder()
            .withPlaceholder("Password")
            .withPosition(10, 10)
            .withSize(200, 30)
            .masked(true)
            .withMaskCharacter('*')
            .build();
    }
    
    public static UITextField createNumberField() {
        return new UITextFieldBuilder()
            .withPlaceholder("Enter number")
            .withPosition(10, 10)
            .withSize(200, 30)
            .numericOnly(true)
            .withValidator(text -> {
                try {
                    int value = Integer.parseInt(text);
                    return value >= 0 && value <= 100;
                } catch (NumberFormatException e) {
                    return false;
                }
            })
            .build();
    }
}
```

### Lists and ScrollViews

```java
public class ListExamples {
    public static UIList createSimpleList() {
        UIList list = new UIListBuilder()
            .withPosition(10, 10)
            .withSize(300, 200)
            .withItemHeight(30)
            .selectable(true)
            .multiSelect(false)
            .build();
        
        list.addItem(new UIListItem("Item 1"));
        list.addItem(new UIListItem("Item 2"));
        list.addItem(new UIListItem("Item 3"));
        
        list.onItemSelected(item -> {
            System.out.println("Selected: " + item.getText());
        });
        
        return list;
    }
    
    public static UIList createQuestList(List<Quest> quests) {
        UIList list = new UIListBuilder()
            .withPosition(10, 50)
            .withSize(400, 300)
            .withItemHeight(60)
            .build();
        
        for (Quest quest : quests) {
            UIListItem item = new UIListItem(quest.getName());
            item.setDescription(quest.getDescription());
            item.setIcon(quest.getIcon());
            item.setData("quest", quest);
            list.addItem(item);
        }
        
        list.onItemSelected(item -> {
            Quest quest = (Quest) item.getData("quest");
            showQuestDetails(quest);
        });
        
        return list;
    }
    
    public static UIScrollView createScrollView() {
        UIScrollView scrollView = new UIScrollViewBuilder()
            .withPosition(10, 10)
            .withSize(400, 300)
            .withContentSize(400, 1000)
            .verticalScrollbar(true)
            .horizontalScrollbar(false)
            .build();
        
        // Add content
        for (int i = 0; i < 20; i++) {
            UILabel label = new UILabelBuilder()
                .withText("Item " + (i + 1))
                .withPosition(10, 10 + i * 30)
                .withSize(380, 25)
                .build();
            scrollView.addContent(label);
        }
        
        return scrollView;
    }
}
```

### Checkboxes and Radio Buttons

```java
public class SelectionExamples {
    public static UICheckbox createCheckbox() {
        return new UICheckboxBuilder()
            .withLabel("Accept Terms")
            .withPosition(10, 10)
            .withSize(20, 20)
            .checked(false)
            .onChange(checked -> {
                System.out.println("Checkbox: " + checked);
            })
            .build();
    }
    
    public static UIRadioGroup createRadioGroup() {
        UIRadioGroup group = new UIRadioGroup();
        
        UIRadioButton option1 = new UIRadioButtonBuilder()
            .withLabel("Option 1")
            .withPosition(10, 10)
            .withGroup(group)
            .build();
        
        UIRadioButton option2 = new UIRadioButtonBuilder()
            .withLabel("Option 2")
            .withPosition(10, 40)
            .withGroup(group)
            .build();
        
        UIRadioButton option3 = new UIRadioButtonBuilder()
            .withLabel("Option 3")
            .withPosition(10, 70)
            .withGroup(group)
            .build();
        
        group.onSelectionChanged(selected -> {
            System.out.println("Selected: " + selected.getLabel());
        });
        
        return group;
    }
}
```

### Progress Bars

```java
public class ProgressBarExamples {
    public static UIProgressBar createSimpleProgressBar() {
        return new UIProgressBarBuilder()
            .withPosition(10, 10)
            .withSize(300, 25)
            .withProgress(0.5f)  // 50%
            .withBackgroundColor(0x333333)
            .withFillColor(0x4CAF50)
            .withBorderColor(0x666666)
            .showPercentage(true)
            .build();
    }
    
    public static UIProgressBar createQuestProgressBar(Quest quest, Player player) {
        UIProgressBar bar = new UIProgressBarBuilder()
            .withPosition(10, 10)
            .withSize(400, 30)
            .withBackgroundColor(0x222222)
            .withFillColor(0xFFD700)
            .showPercentage(true)
            .build();
        
        // Update based on quest progress
        bar.setProgressProvider(() -> {
            return quest.getProgress(player);
        });
        
        return bar;
    }
    
    public static UIProgressBar createAnimatedProgressBar() {
        UIProgressBar bar = new UIProgressBarBuilder()
            .withPosition(10, 10)
            .withSize(300, 25)
            .animated(true)
            .animationSpeed(1.0f)
            .build();
        
        return bar;
    }
}
```

---

## Layout Systems

### Absolute Layout

```java
public class AbsoluteLayoutExample {
    public static UIPanel createAbsoluteLayout() {
        UIPanel panel = new UIPanel();
        panel.setSize(400, 300);
        
        // Components positioned absolutely
        UILabel title = new UILabelBuilder()
            .withText("Title")
            .withPosition(150, 10)
            .withSize(100, 30)
            .build();
        
        UIButton button1 = new UIButtonBuilder()
            .withText("Button 1")
            .withPosition(50, 100)
            .withSize(100, 30)
            .build();
        
        UIButton button2 = new UIButtonBuilder()
            .withText("Button 2")
            .withPosition(250, 100)
            .withSize(100, 30)
            .build();
        
        panel.addChild(title);
        panel.addChild(button1);
        panel.addChild(button2);
        
        return panel;
    }
}
```

### Vertical Layout

```java
public class VerticalLayoutExample {
    public static UIPanel createVerticalLayout() {
        UIPanel panel = new UIPanel();
        panel.setLayout(new VerticalLayout()
            .withSpacing(10)
            .withPadding(10)
            .withAlignment(VerticalAlignment.TOP)
        );
        
        panel.addChild(new UILabelBuilder()
            .withText("Item 1")
            .withSize(200, 30)
            .build());
        
        panel.addChild(new UILabelBuilder()
            .withText("Item 2")
            .withSize(200, 30)
            .build());
        
        panel.addChild(new UILabelBuilder()
            .withText("Item 3")
            .withSize(200, 30)
            .build());
        
        return panel;
    }
}
```

### Horizontal Layout

```java
public class HorizontalLayoutExample {
    public static UIPanel createHorizontalLayout() {
        UIPanel panel = new UIPanel();
        panel.setLayout(new HorizontalLayout()
            .withSpacing(10)
            .withPadding(10)
            .withAlignment(HorizontalAlignment.LEFT)
        );
        
        panel.addChild(new UIButtonBuilder()
            .withText("Button 1")
            .withSize(100, 30)
            .build());
        
        panel.addChild(new UIButtonBuilder()
            .withText("Button 2")
            .withSize(100, 30)
            .build());
        
        panel.addChild(new UIButtonBuilder()
            .withText("Button 3")
            .withSize(100, 30)
            .build());
        
        return panel;
    }
}
```

### Grid Layout

```java
public class GridLayoutExample {
    public static UIPanel createGridLayout() {
        UIPanel panel = new UIPanel();
        panel.setLayout(new GridLayout()
            .withColumns(3)
            .withRows(3)
            .withSpacing(5)
            .withPadding(10)
        );
        
        for (int i = 0; i < 9; i++) {
            panel.addChild(new UIButtonBuilder()
                .withText("Item " + (i + 1))
                .withSize(80, 80)
                .build());
        }
        
        return panel;
    }
    
    public static UIPanel createInventoryGrid() {
        UIPanel panel = new UIPanel();
        panel.setLayout(new GridLayout()
            .withColumns(9)
            .withRows(4)
            .withSpacing(2)
        );
        
        for (int i = 0; i < 36; i++) {
            UIItemSlot slot = new UIItemSlot();
            slot.setSize(40, 40);
            panel.addChild(slot);
        }
        
        return panel;
    }
}
```

### Anchor Layout

```java
public class AnchorLayoutExample {
    public static UIPanel createAnchorLayout() {
        UIPanel panel = new UIPanel();
        panel.setSize(400, 300);
        
        // Top-left anchored
        UIButton topLeft = new UIButtonBuilder()
            .withText("Top Left")
            .withSize(100, 30)
            .build();
        topLeft.setAnchor(Anchor.TOP_LEFT);
        topLeft.setOffset(10, 10);
        
        // Top-right anchored
        UIButton topRight = new UIButtonBuilder()
            .withText("Top Right")
            .withSize(100, 30)
            .build();
        topRight.setAnchor(Anchor.TOP_RIGHT);
        topRight.setOffset(-110, 10);
        
        // Bottom-center anchored
        UIButton bottomCenter = new UIButtonBuilder()
            .withText("Bottom Center")
            .withSize(100, 30)
            .build();
        bottomCenter.setAnchor(Anchor.BOTTOM_CENTER);
        bottomCenter.setOffset(-50, -40);
        
        panel.addChild(topLeft);
        panel.addChild(topRight);
        panel.addChild(bottomCenter);
        
        return panel;
    }
}
```

---

## Event Handling

### Mouse Events

```java
public class MouseEventExamples {
    public static void setupMouseEvents(UIComponent component) {
        // Click events
        component.onClick(event -> {
            System.out.println("Clicked at: " + event.getX() + ", " + event.getY());
            
            if (event.isLeftClick()) {
                System.out.println("Left click");
            } else if (event.isRightClick()) {
                System.out.println("Right click");
            }
        });
        
        // Hover events
        component.onMouseEnter(event -> {
            component.setBackgroundColor(0x444444);
        });
        
        component.onMouseLeave(event -> {
            component.setBackgroundColor(0x333333);
        });
        
        // Drag events
        component.onDragStart(event -> {
            System.out.println("Drag started");
        });
        
        component.onDrag(event -> {
            component.setPosition(event.getX(), event.getY());
        });
        
        component.onDragEnd(event -> {
            System.out.println("Drag ended");
        });
        
        // Mouse wheel
        component.onMouseWheel(event -> {
            int delta = event.getWheelDelta();
            System.out.println("Wheel delta: " + delta);
        });
    }
}
```

### Keyboard Events

```java
public class KeyboardEventExamples {
    public static void setupKeyboardEvents(UIComponent component) {
        component.onKeyPress(event -> {
            System.out.println("Key pressed: " + event.getKey());
            
            if (event.isControl() && event.getKey() == Key.S) {
                System.out.println("Ctrl+S pressed");
                event.consume();
            }
        });
        
        component.onKeyRelease(event -> {
            System.out.println("Key released: " + event.getKey());
        });
        
        component.onKeyType(event -> {
            char character = event.getCharacter();
            System.out.println("Character typed: " + character);
        });
    }
}
```

### Custom Events

```java
public class CustomEventExamples {
    public static class QuestAcceptedEvent extends UIEvent {
        private final Quest quest;
        
        public QuestAcceptedEvent(Quest quest) {
            super("quest_accepted");
            this.quest = quest;
        }
        
        public Quest getQuest() {
            return quest;
        }
    }
    
    public static void setupCustomEvents(UIComponent component) {
        component.addEventListener("quest_accepted", event -> {
            if (event instanceof QuestAcceptedEvent) {
                QuestAcceptedEvent qEvent = (QuestAcceptedEvent) event;
                System.out.println("Quest accepted: " + qEvent.getQuest().getName());
            }
        });
        
        // Dispatch custom event
        component.dispatchEvent(new QuestAcceptedEvent(someQuest));
    }
}
```

---

## Styling and Theming

### Basic Styling

```java
public class StylingExamples {
    public static void styleButton(UIButton button) {
        button.setBackgroundColor(0x4CAF50);
        button.setForegroundColor(0xFFFFFF);
        button.setBorderColor(0x45A049);
        button.setBorderWidth(2);
        button.setCornerRadius(5);
        button.setPadding(10, 5, 10, 5);
        button.setFont("sans-serif");
        button.setFontSize(14);
        button.setBold(false);
        button.setItalic(false);
    }
    
    public static void stylePanel(UIPanel panel) {
        panel.setBackgroundColor(0x222222);
        panel.setBorderColor(0x444444);
        panel.setBorderWidth(1);
        panel.setCornerRadius(0);
        panel.setPadding(10);
        panel.setShadow(true);
        panel.setShadowColor(0x000000);
        panel.setShadowOffset(2, 2);
        panel.setShadowBlur(5);
    }
}
```

### Theme System

```java
public class ThemeSystem {
    public static class Theme {
        private Map<String, Object> properties = new HashMap<>();
        
        public void set(String key, Object value) {
            properties.put(key, value);
        }
        
        public <T> T get(String key, T defaultValue) {
            return (T) properties.getOrDefault(key, defaultValue);
        }
    }
    
    public static Theme createDarkTheme() {
        Theme theme = new Theme();
        theme.set("background.primary", 0x1E1E1E);
        theme.set("background.secondary", 0x252526);
        theme.set("border.color", 0x3E3E42);
        theme.set("text.primary", 0xFFFFFF);
        theme.set("text.secondary", 0xCCCCCC);
        theme.set("accent.primary", 0x007ACC);
        theme.set("accent.hover", 0x005A9E);
        theme.set("success.color", 0x4CAF50);
        theme.set("warning.color", 0xFF9800);
        theme.set("error.color", 0xF44336);
        return theme;
    }
    
    public static void applyTheme(UIComponent component, Theme theme) {
        component.setBackgroundColor(theme.get("background.primary", 0x1E1E1E));
        component.setForegroundColor(theme.get("text.primary", 0xFFFFFF));
        component.setBorderColor(theme.get("border.color", 0x3E3E42));
    }
}
```

---

## Quest Tracker Example

### Complete Quest Tracker UI

```java
public class QuestTrackerUI {
    private UIWindow window;
    private UIList questList;
    private UIPanel detailPanel;
    private final QuestManager questManager;
    
    public QuestTrackerUI(QuestManager questManager) {
        this.questManager = questManager;
        createUI();
    }
    
    private void createUI() {
        window = new UIWindowBuilder("quest_tracker")
            .withTitle("Quest Tracker")
            .withSize(500, 600)
            .atPosition(50, 50)
            .closeable(true)
            .draggable(true)
            .build();
        
        // Create main container
        UIPanel container = new UIPanel();
        container.setLayout(new VerticalLayout().withSpacing(10).withPadding(10));
        
        // Add filter buttons
        UIPanel filterPanel = createFilterPanel();
        container.addChild(filterPanel);
        
        // Add quest list
        questList = createQuestList();
        container.addChild(questList);
        
        // Add detail panel
        detailPanel = createDetailPanel();
        container.addChild(detailPanel);
        
        window.setContent(container);
    }
    
    private UIPanel createFilterPanel() {
        UIPanel panel = new UIPanel();
        panel.setLayout(new HorizontalLayout().withSpacing(5));
        panel.setSize(480, 40);
        
        UIButton allButton = new UIButtonBuilder()
            .withText("All")
            .withSize(100, 30)
            .onClick(e -> filterQuests(QuestFilter.ALL))
            .build();
        
        UIButton activeButton = new UIButtonBuilder()
            .withText("Active")
            .withSize(100, 30)
            .onClick(e -> filterQuests(QuestFilter.ACTIVE))
            .build();
        
        UIButton completedButton = new UIButtonBuilder()
            .withText("Completed")
            .withSize(100, 30)
            .onClick(e -> filterQuests(QuestFilter.COMPLETED))
            .build();
        
        panel.addChild(allButton);
        panel.addChild(activeButton);
        panel.addChild(completedButton);
        
        return panel;
    }
    
    private UIList createQuestList() {
        UIList list = new UIListBuilder()
            .withSize(480, 300)
            .withItemHeight(80)
            .build();
        
        list.onItemSelected(item -> {
            Quest quest = (Quest) item.getData("quest");
            showQuestDetails(quest);
        });
        
        return list;
    }
    
    private UIPanel createDetailPanel() {
        UIPanel panel = new UIPanel();
        panel.setSize(480, 200);
        panel.setLayout(new VerticalLayout().withSpacing(5).withPadding(10));
        panel.setBorderColor(0x444444);
        panel.setBorderWidth(1);
        
        return panel;
    }
    
    private void filterQuests(QuestFilter filter) {
        questList.clear();
        
        List<Quest> quests = switch (filter) {
            case ALL -> questManager.getAllQuests();
            case ACTIVE -> questManager.getActiveQuests();
            case COMPLETED -> questManager.getCompletedQuests();
        };
        
        for (Quest quest : quests) {
            UIListItem item = createQuestListItem(quest);
            questList.addItem(item);
        }
    }
    
    private UIListItem createQuestListItem(Quest quest) {
        UIPanel itemPanel = new UIPanel();
        itemPanel.setSize(460, 70);
        
        // Quest icon
        UIImage icon = new UIImage(quest.getIcon());
        icon.setPosition(5, 5);
        icon.setSize(60, 60);
        itemPanel.addChild(icon);
        
        // Quest name
        UILabel name = new UILabelBuilder()
            .withText(quest.getName())
            .withPosition(75, 5)
            .withSize(380, 25)
            .withFontSize(16)
            .bold(true)
            .build();
        itemPanel.addChild(name);
        
        // Quest description
        UILabel desc = new UILabelBuilder()
            .withText(quest.getDescription())
            .withPosition(75, 30)
            .withSize(380, 15)
            .withFontSize(12)
            .withColor(0xCCCCCC)
            .build();
        itemPanel.addChild(desc);
        
        // Progress bar
        UIProgressBar progress = new UIProgressBarBuilder()
            .withPosition(75, 50)
            .withSize(380, 15)
            .withProgress(quest.getProgress())
            .showPercentage(true)
            .build();
        itemPanel.addChild(progress);
        
        UIListItem item = new UIListItem(itemPanel);
        item.setData("quest", quest);
        
        return item;
    }
    
    private void showQuestDetails(Quest quest) {
        detailPanel.clear();
        
        // Quest name
        UILabel name = new UILabelBuilder()
            .withText(quest.getName())
            .withSize(460, 30)
            .withFontSize(18)
            .bold(true)
            .build();
        detailPanel.addChild(name);
        
        // Quest description
        UILabel desc = new UILabelBuilder()
            .withText(quest.getDescription())
            .withSize(460, 40)
            .withFontSize(12)
            .wordWrap(true)
            .build();
        detailPanel.addChild(desc);
        
        // Objectives
        UILabel objectivesTitle = new UILabelBuilder()
            .withText("Objectives:")
            .withSize(460, 20)
            .bold(true)
            .build();
        detailPanel.addChild(objectivesTitle);
        
        for (QuestObjective objective : quest.getObjectives()) {
            UIPanel objPanel = new UIPanel();
            objPanel.setSize(460, 25);
            objPanel.setLayout(new HorizontalLayout().withSpacing(5));
            
            UICheckbox checkbox = new UICheckboxBuilder()
                .withSize(20, 20)
                .checked(objective.isComplete())
                .enabled(false)
                .build();
            
            UILabel objText = new UILabelBuilder()
                .withText(objective.getProgressText())
                .withSize(430, 20)
                .build();
            
            objPanel.addChild(checkbox);
            objPanel.addChild(objText);
            detailPanel.addChild(objPanel);
        }
        
        // Rewards
        if (!quest.getRewards().isEmpty()) {
            UILabel rewardsTitle = new UILabelBuilder()
                .withText("Rewards:")
                .withSize(460, 20)
                .bold(true)
                .build();
            detailPanel.addChild(rewardsTitle);
            
            // Display rewards...
        }
    }
    
    public void show() {
        filterQuests(QuestFilter.ALL);
        window.show();
    }
    
    public void hide() {
        window.hide();
    }
    
    private enum QuestFilter {
        ALL, ACTIVE, COMPLETED
    }
}
```

---

## Custom Components

### Creating Custom Components

```java
public class CustomProgressWheel extends UIComponent {
    private float progress = 0.0f;
    private int backgroundColor = 0x333333;
    private int fillColor = 0x4CAF50;
    private int size = 100;
    
    @Override
    public void render(RenderContext ctx) {
        // Draw background circle
        ctx.drawCircle(
            getBounds().getCenterX(),
            getBounds().getCenterY(),
            size / 2,
            backgroundColor
        );
        
        // Draw progress arc
        float angle = progress * 360;
        ctx.drawArc(
            getBounds().getCenterX(),
            getBounds().getCenterY(),
            size / 2,
            0,
            angle,
            fillColor
        );
        
        // Draw percentage text
        String text = String.format("%.0f%%", progress * 100);
        ctx.drawText(
            text,
            getBounds().getCenterX(),
            getBounds().getCenterY(),
            TextAlignment.CENTER
        );
    }
    
    @Override
    public void update(float deltaTime) {
        // Update logic
    }
    
    @Override
    public void handleEvent(UIEvent event) {
        // Event handling
    }
    
    public void setProgress(float progress) {
        this.progress = Math.max(0, Math.min(1, progress));
    }
    
    public float getProgress() {
        return progress;
    }
}
```

### Custom Layout Manager

```java
public class CircularLayout implements LayoutManager {
    private double radius = 100;
    private double startAngle = 0;
    
    @Override
    public void layout(UIContainer container) {
        List<UIComponent> children = container.getChildren();
        int count = children.size();
        
        if (count == 0) return;
        
        double angleStep = (2 * Math.PI) / count;
        Point center = container.getBounds().getCenter();
        
        for (int i = 0; i < count; i++) {
            double angle = startAngle + (i * angleStep);
            int x = (int) (center.x + radius * Math.cos(angle));
            int y = (int) (center.y + radius * Math.sin(angle));
            
            UIComponent child = children.get(i);
            child.setPosition(x - child.getWidth() / 2, y - child.getHeight() / 2);
        }
    }
}
```

---

## Best Practices

### DO:
- ✅ Use layout managers instead of absolute positioning
- ✅ Implement responsive designs
- ✅ Cache frequently accessed UI elements
- ✅ Use event delegation for lists
- ✅ Dispose unused UI resources
- ✅ Test on different screen resolutions
- ✅ Provide keyboard navigation

### DON'T:
- ❌ Update UI every frame unnecessarily
- ❌ Create excessive nested containers
- ❌ Ignore memory leaks
- ❌ Hard-code sizes and positions
- ❌ Forget accessibility features
- ❌ Overcomplicate UI hierarchy

---

## Performance Optimization

### Rendering Optimization

```java
public class RenderingOptimization {
    // Use dirty flags
    private boolean needsRedraw = true;
    
    @Override
    public void render(RenderContext ctx) {
        if (!needsRedraw) {
            return;
        }
        
        // Render logic
        renderComponent(ctx);
        
        needsRedraw = false;
    }
    
    public void invalidate() {
        needsRedraw = true;
    }
    
    // Batch rendering
    public void renderBatch(RenderContext ctx, List<UIComponent> components) {
        ctx.beginBatch();
        for (UIComponent component : components) {
            component.render(ctx);
        }
        ctx.endBatch();
    }
}
```

### Memory Management

```java
public class MemoryManagement {
    public void disposeWindow(UIWindow window) {
        // Remove event listeners
        window.clearEventListeners();
        
        // Dispose children
        for (UIComponent child : window.getChildren()) {
            disposeComponent(child);
        }
        
        // Clear references
        window.clear();
    }
    
    private void disposeComponent(UIComponent component) {
        if (component instanceof UIContainer) {
            UIContainer container = (UIContainer) component;
            for (UIComponent child : container.getChildren()) {
                disposeComponent(child);
            }
        }
        
        component.dispose();
    }
}
```

---

## Troubleshooting

### UI Not Showing

Check rendering order, visibility flags, and z-index.

### Events Not Firing

Verify event listeners are registered and components are interactive.

### Performance Issues

Profile rendering, reduce nesting, use batching.

For more information:
- [Quest Development](quest-development.md)
- [NPC Integration](npc-integration.md)
- [API Reference](../api/framework-ui.md)
