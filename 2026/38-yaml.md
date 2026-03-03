# Day 38 – YAML Basics

YAML (YAML Ain't Markup Language) is the configuration language used by Docker Compose, Kubernetes, GitHub Actions, GitLab CI, and almost every DevOps tool. Understanding it is non-negotiable.

---

## Task 1: Key-Value Pairs

### person.yaml
```yaml
name: Dikshith Bonu
role: DevOps Engineer
experience_years: 0.5
learning: true
```

### Rules

- **Key-value format:** `key: value`
- **Space after colon:** Required
- **No tabs:** Only spaces for indentation
- **Booleans:** `true`, `false` (lowercase, no quotes)
- **Numbers:** No quotes needed
- **Strings:** Usually no quotes needed (unless special characters)



---

## Task 2: Lists

### Updated person.yaml
```yaml
name: Dikshit Bonu
role: DevOps Engineer
experience_years: 0.5
learning: true

# List format 1: Block style (each item on new line)
tools:
  - Docker
  - Kubernetes
  - Linux
  - Git
  - Bash

# List format 2: Inline style (flow format)
hobbies: [coding, automation, learning DevOps]
```

### Two Ways to Write Lists

**1. Block Style (recommended for readability):**
```yaml
tools:
  - Docker
  - Kubernetes
  - Linux
```

**2. Inline/Flow Style (compact):**
```yaml
hobbies: [coding, automation, learning]
```

**When to use which:**
- Block style: Multi-item lists, better for version control diffs
- Inline style: Short lists (2-3 items), compact configs

---

## Task 3: Nested Objects

### server.yaml
```yaml
server:
  name: production-web-01
  ip: 192.168.1.100
  port: 8080

database:
  host: db.example.com
  name: app_database
  credentials:
    user: admin
    password: supersecret123
```

### Indentation Rules

- **2 spaces per level:** Standard convention
- **Consistent indentation:** All nested items at same level use same indent
- **No tabs:** YAML parsers reject tabs

### Testing with Tabs

**Broken (with tab):**
```yaml
server:
	name: web-01  # TAB used here
```

**Error:**
```
while scanning for the next token
found character '\t' that cannot start any token
```

**YAML absolutely rejects tabs. Always use spaces.**

---

## Task 4: Multi-line Strings

### Updated server.yaml
```yaml
server:
  name: production-web-01
  ip: 192.168.1.100
  port: 8080

# Literal block (|) - preserves newlines
startup_script: |
  #!/bin/bash
  echo "Starting server..."
  systemctl start nginx
  echo "Server started successfully"

# Folded block (>) - folds into single line
description: >
  This is a production web server
  running Nginx on port 8080
  serving the main application.

database:
  host: db.example.com
  name: app_database
  credentials:
    user: admin
    password: supersecret123
```

### Literal (|) vs Folded (>)

**Literal Block (`|`) - Preserves newlines:**
```yaml
script: |
  line 1
  line 2
  line 3
```
**Parsed as:**
```
line 1
line 2
line 3
```

**Folded Block (`>`) - Folds into one line:**
```yaml
description: >
  This is a long
  description that spans
  multiple lines.
```
**Parsed as:**
```
This is a long description that spans multiple lines.
```

**When to use:**
- `|` (literal): Scripts, commands, code blocks, preserve formatting
- `>` (folded): Long descriptions, paragraphs, where line breaks don't matter


---

## Task 6: Spot the Difference

### Block 1 - Correct ✓
```yaml
name: devops
tools:
  - docker
  - kubernetes
```

**Indentation:**
- `tools:` at root level (0 spaces)
- `- docker` indented 2 spaces
- `- kubernetes` indented 2 spaces

---

### Block 2 - Broken ✗
```yaml
name: devops
tools:
- docker
  - kubernetes
```

**What's wrong:**
- First item `- docker` has 0 space indentation
- Second item `- kubernetes` has 2 space indentation
- **Inconsistent indentation!** List items must be at same level

**Error:**
```
while parsing a block mapping
expected <block end>, but found '-'
```

**Fixed version:**
```yaml
name: devops
tools:
  - docker
  - kubernetes
```

All list items indented equally (2 spaces).

---

## YAML Files Created

### person.yaml (Final Version)
```yaml
name: Dikshit Bonu
role: DevOps Engineer
experience_years: 0.5
learning: true

tools:
  - Docker
  - Kubernetes
  - Linux
  - Git
  - Bash

hobbies: [coding, automation, learning DevOps]
```

### server.yaml (Final Version)
```yaml
server:
  name: production-web-01
  ip: 192.168.1.100
  port: 8080

startup_script: |
  #!/bin/bash
  echo "Starting server..."
  systemctl start nginx
  echo "Server started successfully"

description: >
  This is a production web server
  running Nginx on port 8080
  serving the main application.

database:
  host: db.example.com
  name: app_database
  credentials:
    user: admin
    password: supersecret123
```


---

## YAML Syntax Reference

### Data Types
```yaml
# String (no quotes needed usually)
name: devops

# String with special characters (needs quotes)
message: "Error: failed"
path: "/home/user"

# Number
age: 25
price: 19.99

# Boolean
enabled: true
disabled: false

# Null
value: null
empty: ~
```

### Lists
```yaml
# Block style
fruits:
  - apple
  - banana
  - orange

# Inline style
colors: [red, green, blue]

# Nested lists
matrix:
  - [1, 2, 3]
  - [4, 5, 6]
```

### Dictionaries (Objects)
```yaml
# Simple
person:
  name: John
  age: 30

# Nested
company:
  name: TechCorp
  address:
    street: 123 Main St
    city: NYC
    zip: 10001
```

### Multi-line Strings
```yaml
# Literal (preserves newlines)
script: |
  echo "Line 1"
  echo "Line 2"

# Folded (folds to one line)
description: >
  This is a very long
  description that will be
  folded into a single line.

# Quoted multi-line
text: "First line\nSecond line"
```

### Comments
```yaml
# This is a comment
name: value  # Inline comment

# Multi-line comment:
# Line 1
# Line 2
key: value
```


---

## Common YAML Mistakes

### 1. Using Tabs
```yaml
# WRONG 
server:
    name: web  # Tab used
	
# RIGHT 
server:
  name: web  # 2 spaces
```

### 2. Inconsistent Indentation
```yaml
# WRONG 
items:
  - one
   - two  # 3 spaces instead of 2

# RIGHT 
items:
  - one
  - two  # Consistent 2 spaces
```

### 3. Missing Space After Colon
```yaml
# WRONG 
name:value

# RIGHT 
name: value
```

### 4. Incorrect Boolean
```yaml
# WRONG 
enabled: True   # Capital T
active: "true"  # String, not boolean

# RIGHT 
enabled: true
active: false
```

### 5. Quoting Numbers
```yaml
# WRONG 
port: "8080"  # String, not number

# RIGHT 
port: 8080
```

---

## YAML Best Practices

1. **Use 2 spaces for indentation** (industry standard)
2. **Never use tabs** (YAML spec forbids them)
3. **Be consistent with style** (block vs inline lists)
4. **Use quotes when needed** (special chars, preserve type)
5. **Comment complex configs** (explain non-obvious parts)
6. **Validate before using** (catch syntax errors early)
7. **Use meaningful keys** (descriptive names)
8. **Keep it readable** (humans read YAML more than machines)

---


## Key Takeaways

**3 Most Important Points:**

1. **Indentation is Everything**
   - YAML structure defined by indentation
   - Must use spaces (never tabs)
   - Consistent spacing (2 spaces standard)

2. **Simple Syntax, Strict Rules**
   - Readable and human-friendly
   - But unforgiving of syntax errors
   - Always validate before using

3. **Foundation for DevOps**
   - Every CI/CD tool uses YAML
   - Docker Compose, Kubernetes, GitHub Actions
   - Master YAML = master DevOps configs

**YAML is not optional in DevOps. It's the language of infrastructure.**

---
