# UI Framework API Reference

**Package:** `com.argonathsystems.framework.ui`  
**Module ID:** `ui`  
**Version:** 1.0.0  
**Dependencies:** `core`, `config`

## Overview

The UI Framework provides a comprehensive system for creating interactive user interfaces with windows, buttons, labels, text fields, and custom components. Features include layout management, event handling, theming, and responsive design.

## Table of Contents

- [UIComponent](#uicomponent)
- [Window](#window)
- [Button](#button)
- [Label](#label)
- [TextField](#textfield)
- [Container](#container)
- [Layout](#layout)
- [Event Handlers](#event-handlers)
- [Theme System](#theme-system)

---

## UIComponent

Base class for all UI components.

### Class Declaration

```java
public abstract class UIComponent {
    protected String id;
    protected int x, y;
    protected int width, height;
    protected boolean visible = true;
    protected boolean enabled = true;
    protected UIComponent parent;
    
    // Core methods
    public abstract void render(Graphics graphics);
    public abstract void update(float deltaTime);
    
    // Getters/Setters
    public String getId();
    public void setId(String id);
    public int getX();
    public int getY();
    public void setPosition(int x, int y);
    public int getWidth();
    public int getHeight();
    public void setSize(int width, int height);
    public boolean isVisible();
    public void setVisible(boolean visible);
    public boolean isEnabled();
    public void setEnabled(boolean enabled);
    public UIComponent getParent();
    
    // Event methods
    public void onClick(MouseEvent event);
    public void onHover(MouseEvent event);
    public void onFocusGained();
    public void onFocusLost();
    
    // Utility methods
    public boolean contains(int x, int y);
    public void requestFocus();
    public boolean hasFocus();
}
```

### Methods

#### `setPosition(int x, int y)`

Sets the component position.

```java
UIComponent button = new Button("Click Me");
button.setPosition(100, 50);
```

**Parameters:**
- `x` - X coordinate (pixels from left)
- `y` - Y coordinate (pixels from top)

**Coordinate System:** Top-left origin  
**Relative To:** Parent component or window

---

#### `setSize(int width, int height)`

Sets the component size.

```java
button.setSize(120, 40);
```

**Parameters:**
- `width` - Width in pixels
- `height` - Height in pixels

**Constraints:** Minimum size enforced by component type

---

#### `setVisible(boolean visible)`

Shows or hides the component.

```java
button.setVisible(false); // Hide button
```

**Parameters:**
- `visible` - Visibility flag

**Side Effects:** Hidden components don't receive events

---

#### `setEnabled(boolean enabled)`

Enables or disables the component.

```java
button.setEnabled(false); // Disable button
```

**Parameters:**
- `enabled` - Enabled flag

**Appearance:** Disabled components appear grayed out  
**Interaction:** Disabled components don't respond to input

---

#### `contains(int x, int y)`

Checks if a point is within the component bounds.

```java
boolean isInside = button.contains(mouseX, mouseY);
```

**Parameters:**
- `x` - X coordinate
- `y` - Y coordinate

**Returns:** `true` if point is inside component  
**Uses:** Hit testing for mouse events

---

### Example Usage

```java
public class CustomComponent extends UIComponent {
    private String text;
    private Color backgroundColor;
    
    public CustomComponent(String text) {
        this.text = text;
        this.backgroundColor = Color.BLUE;
        setSize(100, 30);
    }
    
    @Override
    public void render(Graphics graphics) {
        if (!isVisible()) return;
        
        // Draw background
        graphics.setColor(backgroundColor);
        graphics.fillRect(x, y, width, height);
        
        // Draw border
        graphics.setColor(Color.BLACK);
        graphics.drawRect(x, y, width, height);
        
        // Draw text
        graphics.setColor(Color.WHITE);
        graphics.drawString(text, x + 10, y + 20);
    }
    
    @Override
    public void update(float deltaTime) {
        // Update logic
    }
    
    @Override
    public void onClick(MouseEvent event) {
        if (!isEnabled()) return;
        logger.info("Custom component clicked: {}", id);
    }
}
```

---

## Window

Container for UI components representing a window.

### Class Declaration

```java
public class Window extends Container {
    private String title;
    private boolean resizable = true;
    private boolean draggable = true;
    private boolean closeable = true;
    private WindowState state = WindowState.NORMAL;
    
    public Window(String title, int width, int height);
    
    // Window management
    public void open();
    public void close();
    public void minimize();
    public void maximize();
    public void restore();
    
    // Getters/Setters
    public String getTitle();
    public void setTitle(String title);
    public boolean isResizable();
    public void setResizable(boolean resizable);
    public boolean isDraggable();
    public void setDraggable(boolean draggable);
    public WindowState getState();
    
    // Event handlers
    public void onClose();
    public void onMinimize();
    public void onMaximize();
}
```

### Methods

#### Constructor

```java
Window questWindow = new Window("Quest Log", 600, 400);
```

**Parameters:**
- `title` - Window title
- `width` - Initial width
- `height` - Initial height

---

#### `open()`

Opens and displays the window.

```java
questWindow.open();
```

**Events:** Fires `WindowOpenEvent`  
**Side Effects:** Shows window, brings to front

---

#### `close()`

Closes the window.

```java
questWindow.close();
```

**Events:** Fires `WindowCloseEvent`  
**Validation:** Can be cancelled via event listener

---

#### `setTitle(String title)`

Sets the window title.

```java
questWindow.setTitle("Quest Log - 5 Active");
```

**Parameters:**
- `title` - New window title

**Display:** Updates title bar immediately

---

### Example Usage

```java
public class QuestLogWindow {
    public static Window createQuestLog(Player player, QuestManager questManager) {
        Window window = new Window("Quest Log", 600, 500);
        window.setPosition(100, 100);
        window.setResizable(true);
        window.setDraggable(true);
        
        // Add title label
        Label titleLabel = new Label("Active Quests");
        titleLabel.setFont(Font.BOLD, 18);
        titleLabel.setPosition(20, 20);
        window.add(titleLabel);
        
        // Add quest list
        int yOffset = 60;
        Collection<QuestProgress> activeQuests = questManager.getActiveQuests(player);
        
        for (QuestProgress progress : activeQuests) {
            Quest quest = progress.getQuest();
            
            // Quest name
            Label questLabel = new Label(quest.getName());
            questLabel.setPosition(20, yOffset);
            window.add(questLabel);
            
            // Progress bar
            ProgressBar progressBar = new ProgressBar();
            progressBar.setPosition(20, yOffset + 25);
            progressBar.setSize(400, 20);
            progressBar.setValue(progress.getCompletionPercentage());
            window.add(progressBar);
            
            // View button
            Button viewButton = new Button("View");
            viewButton.setPosition(440, yOffset + 20);
            viewButton.setSize(80, 30);
            viewButton.setOnClick(event -> showQuestDetails(player, quest));
            window.add(viewButton);
            
            yOffset += 70;
        }
        
        // Add close button
        Button closeButton = new Button("Close");
        closeButton.setPosition(window.getWidth() - 100, window.getHeight() - 50);
        closeButton.setSize(80, 30);
        closeButton.setOnClick(event -> window.close());
        window.add(closeButton);
        
        return window;
    }
}
```

---

## Button

Interactive button component.

### Class Declaration

```java
public class Button extends UIComponent {
    private String text;
    private Icon icon;
    private ButtonStyle style = ButtonStyle.DEFAULT;
    private Consumer<MouseEvent> onClick;
    private boolean pressed = false;
    
    public Button(String text);
    public Button(String text, Icon icon);
    
    // Getters/Setters
    public String getText();
    public void setText(String text);
    public Icon getIcon();
    public void setIcon(Icon icon);
    public ButtonStyle getStyle();
    public void setStyle(ButtonStyle style);
    
    // Event handling
    public void setOnClick(Consumer<MouseEvent> onClick);
    public void setOnHover(Consumer<MouseEvent> onHover);
}
```

### Button Styles

```java
public enum ButtonStyle {
    DEFAULT,      // Standard button
    PRIMARY,      // Primary action (blue)
    SUCCESS,      // Success action (green)
    WARNING,      // Warning action (yellow)
    DANGER,       // Destructive action (red)
    LINK          // Link-style button
}
```

### Methods

#### Constructor

```java
Button button = new Button("Click Me");
Button iconButton = new Button("Save", Icons.SAVE);
```

**Parameters:**
- `text` - Button text
- `icon` - Optional icon

---

#### `setOnClick(Consumer<MouseEvent> onClick)`

Sets the click event handler.

```java
button.setOnClick(event -> {
    player.sendMessage("Button clicked!");
});
```

**Parameters:**
- `onClick` - Click handler

**Event:** Called when button is clicked

---

#### `setStyle(ButtonStyle style)`

Sets the button visual style.

```java
Button deleteButton = new Button("Delete");
deleteButton.setStyle(ButtonStyle.DANGER);
```

**Parameters:**
- `style` - Button style

**Appearance:** Changes colors and styling

---

### Example Usage

```java
public class DialogButtons {
    public static void addQuestAcceptButtons(Window window, Quest quest) {
        // Accept button
        Button acceptButton = new Button("Accept Quest");
        acceptButton.setStyle(ButtonStyle.SUCCESS);
        acceptButton.setSize(150, 40);
        acceptButton.setPosition(100, 300);
        acceptButton.setOnClick(event -> {
            questManager.startQuest(player, quest.getId());
            player.sendMessage("§aQuest started!");
            window.close();
        });
        window.add(acceptButton);
        
        // Decline button
        Button declineButton = new Button("Decline");
        declineButton.setStyle(ButtonStyle.WARNING);
        declineButton.setSize(150, 40);
        declineButton.setPosition(270, 300);
        declineButton.setOnClick(event -> {
            player.sendMessage("Quest declined.");
            window.close();
        });
        window.add(declineButton);
    }
}
```

---

## Label

Text display component.

### Class Declaration

```java
public class Label extends UIComponent {
    private String text;
    private Font font;
    private Color textColor;
    private TextAlignment alignment;
    private boolean wordWrap = false;
    
    public Label(String text);
    
    // Getters/Setters
    public String getText();
    public void setText(String text);
    public Font getFont();
    public void setFont(String fontName, int size);
    public void setFont(FontStyle style, int size);
    public Color getTextColor();
    public void setTextColor(Color color);
    public TextAlignment getAlignment();
    public void setAlignment(TextAlignment alignment);
    public boolean isWordWrap();
    public void setWordWrap(boolean wordWrap);
}
```

### Text Alignment

```java
public enum TextAlignment {
    LEFT,
    CENTER,
    RIGHT,
    JUSTIFY
}
```

### Font Styles

```java
public enum FontStyle {
    REGULAR,
    BOLD,
    ITALIC,
    BOLD_ITALIC
}
```

### Example Usage

```java
// Title label
Label title = new Label("Quest Details");
title.setFont(FontStyle.BOLD, 24);
title.setTextColor(Color.GOLD);
title.setAlignment(TextAlignment.CENTER);
title.setPosition(0, 20);
title.setSize(600, 40);
window.add(title);

// Description label with word wrap
Label description = new Label(quest.getDescription());
description.setFont(FontStyle.REGULAR, 14);
description.setTextColor(Color.WHITE);
description.setWordWrap(true);
description.setPosition(20, 80);
description.setSize(560, 100);
window.add(description);

// Dynamic label (updates)
Label progressLabel = new Label("Progress: 0%");
progressLabel.setPosition(20, 200);
window.add(progressLabel);

// Update label periodically
scheduler.runTaskTimer(() -> {
    QuestProgress progress = questManager.getProgress(player, quest.getId());
    int percentage = progress.getCompletionPercentage();
    progressLabel.setText("Progress: " + percentage + "%");
}, 0, 20); // Update every second
```

---

## TextField

Text input component.

### Class Declaration

```java
public class TextField extends UIComponent {
    private String text = "";
    private String placeholder = "";
    private int maxLength = Integer.MAX_VALUE;
    private boolean password = false;
    private TextFieldValidator validator;
    private Consumer<String> onTextChange;
    
    public TextField();
    public TextField(String placeholder);
    
    // Getters/Setters
    public String getText();
    public void setText(String text);
    public String getPlaceholder();
    public void setPlaceholder(String placeholder);
    public int getMaxLength();
    public void setMaxLength(int maxLength);
    public boolean isPassword();
    public void setPassword(boolean password);
    
    // Validation
    public void setValidator(TextFieldValidator validator);
    public boolean isValid();
    
    // Events
    public void setOnTextChange(Consumer<String> onTextChange);
    public void setOnEnter(Consumer<String> onEnter);
}
```

### Text Field Validator

```java
public interface TextFieldValidator {
    boolean validate(String text);
    String getErrorMessage();
}
```

### Built-in Validators

```java
// Numeric validator
TextFieldValidator numericValidator = new NumericValidator(1, 100);

// Email validator
TextFieldValidator emailValidator = new EmailValidator();

// Custom validator
TextFieldValidator questIdValidator = new TextFieldValidator() {
    @Override
    public boolean validate(String text) {
        return questManager.getRegistry().isRegistered(text);
    }
    
    @Override
    public String getErrorMessage() {
        return "Quest ID not found";
    }
};
```

### Example Usage

```java
public class QuestSearchWindow {
    public static Window createSearchWindow() {
        Window window = new Window("Search Quests", 400, 200);
        
        // Search field
        TextField searchField = new TextField("Enter quest name...");
        searchField.setPosition(20, 50);
        searchField.setSize(360, 30);
        searchField.setMaxLength(50);
        
        // Update search results as user types
        searchField.setOnTextChange(text -> {
            updateSearchResults(window, text);
        });
        
        // Search on Enter key
        searchField.setOnEnter(text -> {
            performSearch(text);
        });
        
        window.add(searchField);
        
        // Search button
        Button searchButton = new Button("Search");
        searchButton.setPosition(300, 90);
        searchButton.setSize(80, 30);
        searchButton.setOnClick(event -> {
            performSearch(searchField.getText());
        });
        window.add(searchButton);
        
        return window;
    }
    
    private static void updateSearchResults(Window window, String query) {
        // Filter and display matching quests
        Collection<Quest> allQuests = questRegistry.getAllQuests();
        List<Quest> matches = allQuests.stream()
            .filter(q -> q.getName().toLowerCase().contains(query.toLowerCase()))
            .limit(10)
            .collect(Collectors.toList());
        
        // Update results display (implementation depends on UI structure)
    }
}
```

---

## Container

Component that can contain other components.

### Class Declaration

```java
public class Container extends UIComponent {
    protected List<UIComponent> children = new ArrayList<>();
    protected Layout layout;
    
    // Child management
    public void add(UIComponent component);
    public void remove(UIComponent component);
    public void removeAll();
    public List<UIComponent> getChildren();
    public UIComponent getChild(String id);
    
    // Layout
    public void setLayout(Layout layout);
    public Layout getLayout();
    public void revalidate();
}
```

### Methods

#### `add(UIComponent component)`

Adds a child component.

```java
Container panel = new Container();
panel.add(new Label("Name:"));
panel.add(new TextField());
```

**Parameters:**
- `component` - Component to add

**Side Effects:** Sets component's parent, applies layout

---

#### `setLayout(Layout layout)`

Sets the layout manager.

```java
Container panel = new Container();
panel.setLayout(new VerticalLayout(10)); // 10px spacing
```

**Parameters:**
- `layout` - Layout manager

**See Also:** [Layout](#layout)

---

### Example Usage

```java
public class QuestObjectivePanel extends Container {
    public QuestObjectivePanel(Quest quest, QuestProgress progress) {
        setLayout(new VerticalLayout(5));
        setSize(500, 300);
        
        // Add objective labels
        for (QuestObjective objective : quest.getObjectives()) {
            Container objectiveRow = new Container();
            objectiveRow.setLayout(new HorizontalLayout(10));
            
            // Checkbox/status
            Checkbox checkbox = new Checkbox();
            checkbox.setChecked(progress.isObjectiveComplete(objective.getId()));
            checkbox.setEnabled(false);
            objectiveRow.add(checkbox);
            
            // Objective description
            Label description = new Label(objective.getDescription());
            objectiveRow.add(description);
            
            // Progress (if applicable)
            if (objective.getAmount() > 1) {
                int current = progress.getObjectiveProgress(objective.getId());
                int required = objective.getAmount();
                Label progressLabel = new Label(
                    String.format("(%d/%d)", current, required)
                );
                progressLabel.setTextColor(
                    current >= required ? Color.GREEN : Color.YELLOW
                );
                objectiveRow.add(progressLabel);
            }
            
            add(objectiveRow);
        }
    }
}
```

---

## Layout

Interface for layout managers.

### Layout Managers

#### VerticalLayout

Arranges components vertically.

```java
Container panel = new Container();
panel.setLayout(new VerticalLayout(10)); // 10px spacing between components

panel.add(new Label("Quest Name:"));
panel.add(new TextField());
panel.add(new Label("Description:"));
panel.add(new TextArea());
panel.add(new Button("Submit"));

// Components automatically positioned vertically with 10px gaps
```

---

#### HorizontalLayout

Arranges components horizontally.

```java
Container buttonBar = new Container();
buttonBar.setLayout(new HorizontalLayout(5)); // 5px spacing

buttonBar.add(new Button("Accept"));
buttonBar.add(new Button("Decline"));
buttonBar.add(new Button("Info"));
```

---

#### GridLayout

Arranges components in a grid.

```java
Container grid = new Container();
grid.setLayout(new GridLayout(3, 2)); // 3 rows, 2 columns

grid.add(new Label("Name:"));
grid.add(new TextField());
grid.add(new Label("Level:"));
grid.add(new TextField());
grid.add(new Label("Category:"));
grid.add(new DropDown());
```

---

#### BorderLayout

Divides container into regions.

```java
Container mainPanel = new Container();
mainPanel.setLayout(new BorderLayout());

mainPanel.add(new Label("Title"), BorderLayout.NORTH);
mainPanel.add(new QuestListPanel(), BorderLayout.WEST);
mainPanel.add(new QuestDetailPanel(), BorderLayout.CENTER);
mainPanel.add(new ButtonBar(), BorderLayout.SOUTH);
```

**Regions:**
- `NORTH` - Top
- `SOUTH` - Bottom
- `EAST` - Right
- `WEST` - Left
- `CENTER` - Middle (expands to fill)

---

#### AbsoluteLayout

No automatic positioning (manual x, y coordinates).

```java
Container panel = new Container();
panel.setLayout(new AbsoluteLayout());

Label label = new Label("Custom Position");
label.setPosition(100, 50);
panel.add(label);
```

---

### Custom Layout Example

```java
public class QuestCardLayout implements Layout {
    private int columns;
    private int cardWidth;
    private int cardHeight;
    private int spacing;
    
    public QuestCardLayout(int columns, int cardWidth, int cardHeight, int spacing) {
        this.columns = columns;
        this.cardWidth = cardWidth;
        this.cardHeight = cardHeight;
        this.spacing = spacing;
    }
    
    @Override
    public void layoutContainer(Container container) {
        List<UIComponent> children = container.getChildren();
        
        int row = 0;
        int col = 0;
        
        for (UIComponent child : children) {
            int x = col * (cardWidth + spacing);
            int y = row * (cardHeight + spacing);
            
            child.setPosition(x, y);
            child.setSize(cardWidth, cardHeight);
            
            col++;
            if (col >= columns) {
                col = 0;
                row++;
            }
        }
    }
}

// Usage
Container questGrid = new Container();
questGrid.setLayout(new QuestCardLayout(3, 200, 150, 10));

for (Quest quest : availableQuests) {
    questGrid.add(new QuestCard(quest));
}
```

---

## Event Handlers

### Mouse Events

```java
public class MouseEvent {
    private int x, y;
    private MouseButton button;
    private boolean pressed;
    private boolean released;
    private int clickCount;
    
    public int getX();
    public int getY();
    public MouseButton getButton();
    public boolean isPressed();
    public boolean isReleased();
    public int getClickCount();
}
```

### Keyboard Events

```java
public class KeyEvent {
    private KeyCode key;
    private boolean pressed;
    private boolean released;
    private boolean shiftDown;
    private boolean ctrlDown;
    private boolean altDown;
    
    public KeyCode getKey();
    public boolean isPressed();
    public boolean isReleased();
    public boolean isShiftDown();
    public boolean isCtrlDown();
    public boolean isAltDown();
}
```

### Event Handler Example

```java
public class InteractiveQuestMap extends UIComponent {
    private Map<String, Rectangle> questRegions = new HashMap<>();
    
    @Override
    public void onClick(MouseEvent event) {
        int x = event.getX();
        int y = event.getY();
        
        // Check if click is on a quest region
        for (Map.Entry<String, Rectangle> entry : questRegions.entrySet()) {
            if (entry.getValue().contains(x, y)) {
                String questId = entry.getKey();
                showQuestDetails(questId);
                break;
            }
        }
    }
    
    @Override
    public void onHover(MouseEvent event) {
        int x = event.getX();
        int y = event.getY();
        
        // Highlight hovered quest region
        for (Map.Entry<String, Rectangle> entry : questRegions.entrySet()) {
            Rectangle region = entry.getValue();
            if (region.contains(x, y)) {
                highlightRegion(region);
                
                // Show tooltip
                String questId = entry.getKey();
                Quest quest = questRegistry.getQuest(questId);
                showTooltip(quest.getName(), x, y);
                break;
            }
        }
    }
}
```

---

## Theme System

### Theme

Defines visual styling for UI components.

```java
public class Theme {
    private Map<String, Color> colors = new HashMap<>();
    private Map<String, Font> fonts = new HashMap<>();
    private Map<String, Integer> dimensions = new HashMap<>();
    
    public Color getColor(String key);
    public void setColor(String key, Color color);
    public Font getFont(String key);
    public void setFont(String key, Font font);
    public int getDimension(String key);
    public void setDimension(String key, int value);
}
```

### Built-in Themes

```java
// Dark theme
Theme darkTheme = Theme.DARK;
darkTheme.setColor("background", Color.rgb(30, 30, 30));
darkTheme.setColor("foreground", Color.rgb(200, 200, 200));
darkTheme.setColor("primary", Color.rgb(66, 133, 244));

// Light theme
Theme lightTheme = Theme.LIGHT;
lightTheme.setColor("background", Color.rgb(255, 255, 255));
lightTheme.setColor("foreground", Color.rgb(33, 33, 33));
lightTheme.setColor("primary", Color.rgb(33, 150, 243));

// Apply theme
UIManager.setTheme(darkTheme);
```

### Custom Theme Example

```java
public class QuestTheme extends Theme {
    public QuestTheme() {
        // Colors
        setColor("background", Color.rgb(20, 20, 30));
        setColor("foreground", Color.rgb(220, 220, 220));
        setColor("primary", Color.rgb(255, 215, 0)); // Gold
        setColor("secondary", Color.rgb(192, 192, 192)); // Silver
        setColor("success", Color.rgb(76, 175, 80));
        setColor("warning", Color.rgb(255, 193, 7));
        setColor("danger", Color.rgb(244, 67, 54));
        
        // Fonts
        setFont("title", new Font("Arial", FontStyle.BOLD, 24));
        setFont("heading", new Font("Arial", FontStyle.BOLD, 18));
        setFont("body", new Font("Arial", FontStyle.REGULAR, 14));
        setFont("small", new Font("Arial", FontStyle.REGULAR, 12));
        
        // Dimensions
        setDimension("button-height", 36);
        setDimension("textfield-height", 32);
        setDimension("spacing", 8);
        setDimension("padding", 16);
    }
}

// Apply custom theme
UIManager.setTheme(new QuestTheme());
```

---

## Complete Example: Quest Details Window

```java
public class QuestDetailsWindowFactory {
    public static Window create(Player player, Quest quest, QuestManager questManager) {
        Window window = new Window(quest.getName(), 700, 600);
        window.setPosition(200, 100);
        
        // Apply theme
        Theme theme = new QuestTheme();
        
        // Container for content
        Container content = new Container();
        content.setLayout(new BorderLayout());
        content.setSize(700, 600);
        
        // Header (North)
        Container header = createHeader(quest, theme);
        content.add(header, BorderLayout.NORTH);
        
        // Main content (Center)
        Container main = createMainContent(player, quest, questManager, theme);
        content.add(main, BorderLayout.CENTER);
        
        // Footer with buttons (South)
        Container footer = createFooter(player, quest, questManager, window, theme);
        content.add(footer, BorderLayout.SOUTH);
        
        window.add(content);
        return window;
    }
    
    private static Container createHeader(Quest quest, Theme theme) {
        Container header = new Container();
        header.setLayout(new VerticalLayout(5));
        header.setSize(700, 100);
        
        Label title = new Label(quest.getName());
        title.setFont(theme.getFont("title"));
        title.setTextColor(theme.getColor("primary"));
        header.add(title);
        
        Label category = new Label("Category: " + quest.getCategory());
        category.setFont(theme.getFont("body"));
        header.add(category);
        
        return header;
    }
    
    private static Container createMainContent(Player player, Quest quest, 
                                               QuestManager questManager, Theme theme) {
        Container main = new Container();
        main.setLayout(new VerticalLayout(10));
        
        // Description
        Label descLabel = new Label("Description");
        descLabel.setFont(theme.getFont("heading"));
        main.add(descLabel);
        
        Label description = new Label(quest.getDescription());
        description.setWordWrap(true);
        description.setSize(660, 80);
        main.add(description);
        
        // Objectives
        Label objLabel = new Label("Objectives");
        objLabel.setFont(theme.getFont("heading"));
        main.add(objLabel);
        
        QuestProgress progress = questManager.getProgress(player, quest.getId());
        if (progress != null) {
            main.add(new QuestObjectivePanel(quest, progress));
        } else {
            for (QuestObjective objective : quest.getObjectives()) {
                Label objText = new Label("• " + objective.getDescription());
                main.add(objText);
            }
        }
        
        // Rewards
        Label rewardLabel = new Label("Rewards");
        rewardLabel.setFont(theme.getFont("heading"));
        main.add(rewardLabel);
        
        for (QuestReward reward : quest.getRewards()) {
            Label rewardText = new Label("• " + reward.getDescription());
            rewardText.setTextColor(theme.getColor("success"));
            main.add(rewardText);
        }
        
        return main;
    }
    
    private static Container createFooter(Player player, Quest quest, 
                                          QuestManager questManager, 
                                          Window window, Theme theme) {
        Container footer = new Container();
        footer.setLayout(new HorizontalLayout(10));
        footer.setSize(700, 60);
        
        boolean canStart = questManager.canStart(player, quest.getId());
        boolean isActive = questManager.isActive(player, quest.getId());
        
        if (canStart && !isActive) {
            Button startButton = new Button("Start Quest");
            startButton.setStyle(ButtonStyle.SUCCESS);
            startButton.setSize(150, 40);
            startButton.setOnClick(event -> {
                questManager.startQuest(player, quest.getId());
                window.close();
                player.sendMessage("§aQuest started!");
            });
            footer.add(startButton);
        }
        
        if (isActive) {
            Button abandonButton = new Button("Abandon");
            abandonButton.setStyle(ButtonStyle.DANGER);
            abandonButton.setSize(150, 40);
            abandonButton.setOnClick(event -> {
                // Show confirmation dialog
                showAbandonConfirmation(player, quest, questManager, window);
            });
            footer.add(abandonButton);
            
            Button trackButton = new Button("Track");
            trackButton.setStyle(ButtonStyle.PRIMARY);
            trackButton.setSize(150, 40);
            trackButton.setOnClick(event -> {
                // Enable quest tracking
                player.sendMessage("§eNow tracking: " + quest.getName());
                window.close();
            });
            footer.add(trackButton);
        }
        
        Button closeButton = new Button("Close");
        closeButton.setSize(100, 40);
        closeButton.setOnClick(event -> window.close());
        footer.add(closeButton);
        
        return footer;
    }
}
```

---

## See Also

- [Platform Core API](platform-core.md) - Core platform services
- [Quest Framework API](framework-quest.md) - Quest system integration
- [NPC Framework API](framework-npc.md) - NPC dialog UI
- [Event System API](events.md) - UI event handling
