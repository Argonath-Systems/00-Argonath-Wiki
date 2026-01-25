# Contributing to Argonath Wiki

Thank you for your interest in improving the Argonath Systems documentation! This guide will help you contribute effectively.

## How to Contribute

### Types of Contributions

1. **Fix Typos/Errors** - Correct mistakes in existing documentation
2. **Improve Clarity** - Rewrite confusing sections
3. **Add Examples** - Provide code examples and use cases
4. **New Guides** - Write guides for uncovered topics
5. **Update Documentation** - Keep docs in sync with code changes

### Getting Started

1. **Fork the Repository**
   ```bash
   gh repo fork yourusername/Argonath-Wiki
   ```

2. **Clone Your Fork**
   ```bash
   git clone https://github.com/yourusername/Argonath-Wiki.git
   cd Argonath-Wiki
   ```

3. **Create a Branch**
   ```bash
   git checkout -b docs/your-improvement-name
   ```

4. **Make Your Changes**
   - Edit markdown files
   - Test code examples
   - Preview formatting

5. **Commit and Push**
   ```bash
   git add .
   git commit -m "docs: improve quest integration examples"
   git push origin docs/your-improvement-name
   ```

6. **Open a Pull Request**
   - Go to GitHub and create a PR
   - Describe your changes
   - Link related issues

## Documentation Standards

### Writing Style

- **Clear and Concise** - Use simple language
- **Active Voice** - "Create a quest" not "A quest is created"
- **Present Tense** - "The system uses" not "The system will use"
- **Second Person** - Address the reader as "you"

### Structure

#### Page Template
```markdown
# Page Title

Brief introduction (1-2 paragraphs)

## Section 1

Content...

### Subsection

Content...

## Examples

Code examples...

## Next Steps

- Links to related pages
```

#### Code Examples

Always include:
- **Context** - What the example demonstrates
- **Complete Code** - Runnable if possible
- **Comments** - Explain non-obvious parts
- **Best Practices** - Show the right way

```java
// Good: Complete, commented example
public class QuestExample {
    public void createSimpleQuest() {
        // Create a quest with basic objectives
        Quest quest = questManager.createQuest("example")
            .withName("Example Quest")
            .addObjective(ObjectiveType.COLLECT, "item", 5)
            .build();
    }
}

// Avoid: Incomplete snippets
Quest quest = ...
quest.addObjective(...);
```

### Markdown Formatting

#### Headers
```markdown
# H1 - Page Title (one per page)
## H2 - Major Section
### H3 - Subsection
#### H4 - Minor Point
```

#### Code Blocks
````markdown
```java
// Always specify language
public class Example {
    // Your code here
}
```
````

#### Links
```markdown
# Internal links (relative paths)
See [Architecture Overview](../architecture/overview.md)

# External links
Visit [GitHub](https://github.com)

# Code references
The `QuestManager` class handles quest lifecycle.
```

#### Lists
```markdown
# Ordered for steps
1. First step
2. Second step

# Unordered for items
- Feature A
- Feature B
```

#### Emphasis
```markdown
**Bold** for important terms
*Italic* for emphasis
`code` for class/method names
```

### File Organization

```
docs/
├── getting-started/       # Beginner guides
├── architecture/          # System design
├── guides/                # How-to guides
├── api/                   # API reference
├── integration/           # Integration patterns
├── advanced/              # Advanced topics
├── migration/             # Upgrade guides
└── contributing/          # Contribution docs
```

### Naming Conventions

**Files**: `lowercase-with-hyphens.md`
**Headers**: Title Case for H1/H2, Sentence case for H3+
**Code**: Follow language conventions (camelCase for Java)

## Code Example Guidelines

### Java Code

```java
// ✅ Good: Complete, runnable, commented
import com.hytalemod.argonath.framework.quest.Quest;
import com.hytalemod.argonath.framework.quest.QuestManager;

public class GoodExample {
    private final QuestManager questManager;
    
    public GoodExample(QuestManager questManager) {
        this.questManager = questManager;
    }
    
    /**
     * Creates a simple collection quest.
     * This demonstrates basic quest creation with one objective.
     */
    public Quest createCollectionQuest() {
        return questManager.createQuest("collect_items")
            .withName("Gather Resources")
            .withDescription("Collect 10 wood logs")
            .addObjective(ObjectiveType.COLLECT, "wood_log", 10)
            .addReward("gold", 50)
            .build();
    }
}

// ❌ Bad: Incomplete, no imports, no context
Quest q = qm.create("id")
    .with...
    .build();
```

### Configuration Examples

```yaml
# ✅ Good: Complete with comments
quest:
  # Maximum number of active quests per player
  maxActiveQuests: 5
  
  # Auto-save progress every 5 minutes
  autoSave: true
  saveInterval: 300

# ❌ Bad: No explanations
quest:
  maxActiveQuests: 5
  autoSave: true
```

## Review Process

### Self-Review Checklist

Before submitting:
- [ ] All code examples are tested
- [ ] Links work correctly
- [ ] Spelling/grammar checked
- [ ] Formatting is consistent
- [ ] Examples follow best practices
- [ ] Related docs are updated

### PR Review

Reviewers will check:
1. **Accuracy** - Is the information correct?
2. **Clarity** - Is it easy to understand?
3. **Completeness** - Are all steps included?
4. **Consistency** - Does it match existing docs?

### Response Time

- Initial review: Within 3 days
- Follow-up reviews: Within 1-2 days
- Merge: After approval from 1+ maintainers

## Documentation Types

### Tutorials (Getting Started)
- Step-by-step instructions
- For beginners
- Complete working examples
- Linear progression

### How-To Guides (Guides)
- Goal-oriented
- For intermediate users
- Focus on solving specific problems
- May skip some basics

### Reference (API)
- Technical specifications
- Complete and accurate
- Organized by component
- Less narrative, more facts

### Explanation (Architecture)
- Conceptual understanding
- Why things work the way they do
- Context and background
- Theory and design

## Getting Help

- **Questions**: Open a [Discussion](https://github.com/yourusername/Argonath-Wiki/discussions)
- **Bugs**: Open an [Issue](https://github.com/yourusername/Argonath-Wiki/issues)
- **Chat**: Join our Discord (link in README)

## Recognition

Contributors are recognized in:
- README.md contributors section
- Release notes
- Special thanks in major releases

Thank you for helping make Argonath Systems documentation better! 🎉
