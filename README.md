# Neutral TS - Web Template Engine

## Overview

Neutral Template System is a safe, modular, language-agnostic template engine built in Rust. It can be used as a native Rust library or via IPC for other languages (Python, PHP, Node.js, Go). Templates can be reused across multiple languages with consistent results.

---

## Architecture

### Integration Approaches

- **Rust**: Native library (crate) or IPC client + IPC server
- **Python**: Native package or IPC client + IPC server
- **Other languages** (PHP, Node.js, Go): IPC client + IPC server required

### Client-Server Model (IPC)

The architecture follows a database-like client-server paradigm. Just like a database where different programming languages access data through a client and get identical results, Neutral TS has an IPC server where each programming language has a client. No matter where you run the template, the result will always be the same.

### BIF-Based Block Structure

The main element of Neutral TS is the **BIF (Built-in Function)**, which is the equivalent of functions and displays an output; the output is always a string or nothing (empty string).

Neutral TS is based on BIFs with block structure. We call the set of nested BIFs of the same level a **block**. By design, all BIFs can be nested and there can be a BIF anywhere in another BIF except in the name.

**Block Structure Visualization:**
```
              .-- {:coalesce;
              |       {:code;
              |           {:code; ... :}
              |           {:code; ... :}
    Block --> |           {:code; ... :}
              |       :}
              |       {:code;
              |           {:code; ... :}
              |       :}
              `-- :}

                  {:coalesce;
              .------ {:code;
              |           {:code; ... :}
    Block --> |           {:code; ... :}
              |           {:code; ... :}
              `------ :}
              .------ {:code;
    Block --> |           {:code; ... :}
              `------ :}
                  :}
```

### Data Scope Architecture

There are **global data** and keys available to the entire template, and **local data** and keys available in each block. "Local" information must be preceded by `local::`. Some keys are always "local" such as "snippets", "locale", and others.

---

## Template File Example

```
{:*
   comment
*:}
{:locale; locale.json :}
{:data; local-data.json :}
{:include; theme-snippets.ntpl :}
<!DOCTYPE html>
<html lang="{:lang;:}">
    <head>
        <title>{:trans; Site title :}</title>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        {:snippet; current-theme:head :}
        <link rel="stylesheet" href="bootstrap.min.css">
    </head>
    <body class="{:;body-class:}">
        {:snippet; current-theme:body_begin  :}
        {:snippet; current-theme:body-content :}
        {:snippet; current-theme:body-footer  :}
        <script src="jquery.min.js"></script>
    </body>
</html>
```

---

## Syntax

### BIF (Built-in Function) Structure

The BIF structure follows this pattern:

```
{:[modifiers]name; [flags] params >> code :}
```

Structure breakdown:
```
{:[modifiers]name; [flags] params >> code :}
 │             │ │        │       │   │   │
 │             │ │        │       │   │   └─ Close BIF
 │             │ │        │       │   └─ Code block
 │             │ │        │       └─ Params/Code separator
 │             │ │        └─ Parameters
 │             │ └─ Name separator
 │             └─ BIF name
 └─ Open BIF
```

Visual representation:
```
    .-- open bif
    |   .-- bif name
    |   |      .-- name separator
    |   |      |     .-- params
    |   |      |     |          .-- params/code separator
    |   |      |     |          |    .-- code
    |   |      |     |          |    |      .-- close bif
    v   v      v     v          v    v      v
   -- -----    -  --------     --  -----   --
   {:snippet; snipname    >>   ...        :}
   ------------------------------
              ^ -------------------
              |          ^
              |          |
              |          `-- source
              `-- Built-in function
```

### Modifiers

Modifiers are symbols placed immediately after the opening `{:` and before the BIF name. They alter the behavior of the BIF.

| Modifier | Symbol | Description |
|----------|--------|-------------|
| Upline | `^` | Eliminates previous whitespace |
| Not | `!` | Negates condition or changes behavior |
| Scope | `+` | Adds scope to current level |
| Filter | `&` | Escapes HTML special characters |

Modifiers can be combined: `{:^!defined; varname >> ... :}`

Examples:
```
{:+coalesce; varname >> ... :}   # with modifier "+" (scope)
{:^coalesce; varname >> ... :}   # with modifier "^" (upline)
{:!array; ... :}                 # with modifier "!" (negation)
{:&;varname:}                    # with modifier "&" (filter/escape)
{:^!defined; varname >> ... :}   # combined modifiers
```

### Data Display

Data is defined in JSON format and displayed using the `{:; ... :}` BIF (the variable BIF):

**JSON Data Definition:**
```json
"data": {
    "true": true,
    "false": false,
    "hello": "hello",
    "zero": "0",
    "one": "1",
    "spaces": "   ",
    "empty": "",
    "null": null,
    "array": {
        "hello": "hello",
        "one": "1"
    }
}
```

**Template Usage:**
```
{:;hello:}           <!-- outputs: hello -->
{:;array->hello:}    <!-- outputs: hello -->
{:;array->one:}      <!-- outputs: 1 -->
```

### Variable Access Patterns

#### Simple Variables (`schema.data` - Immutable)
```
{:;varname:}                    # Simple variable
{:;array->key:}                 # Array/object access
{:;array->{:;key:}:}            # Dynamic evaluation
{:;deep->nested->value:}        # Deep nesting access
```

#### Local Variables (`schema.inherit.data` - Mutable)
```
{:;local::varname:}             # Simple local variable
{:;local::array->key:}          # Local array/object access
{:;local::array->{:;key:}:}     # Local dynamic evaluation
```

#### Variable Output Modifiers
```
{:;varname:}      # Output variable (standard)
{:^;varname:}     # Output without preceding whitespace
{:&;varname:}     # Output with HTML escaping
{:!;varname:}     # Output without filtering
```

### Conditional BIFs

Neutral TS provides several conditional BIFs to control content display based on variable states:

#### Defined / Not Defined
Checks if a variable exists in the schema:
```
{:defined; varname >> this shown if varname is defined :}
{:!defined; varname >> this shown if varname is not defined :}
```

#### Filled / Not Filled
Checks if a variable has content (non-empty):
```
{:filled; varname >> this shown if varname has content :}
{:!filled; varname >> this shown if varname is empty :}
```

#### Bool
Checks if a variable evaluates to true/false:
```
{:bool; varname >> this shown if varname is true :}
{:!bool; varname >> this shown if varname is false :}
```

#### Array
Checks if a variable is an array or object:
```
{:array; varname >> this shown if varname is array :}
{:!array; varname >> this shown if varname is not array :}
```

Note: There is no distinction between objects and arrays in Neutral TS.

#### Same / Not Same
String comparison:
```
{:same; /a/b/ >> this shown if a equals b :}
{:!same; /a/b/ >> this shown if a not equals b :}

{:same; /{:;var1:}/{:;var2:}/ >> variables are equal :}
```

#### Contains / Not Contains
Substring checking:
```
{:contains; /haystack/needle/ >> shown if haystack contains needle :}
{:!contains; /haystack/needle/ >> shown if haystack does not contain needle :}

{:contains; /{:;text:}/search/ >> found! :}
```

### Short Circuit Evaluation

Neutral TS implements short circuit evaluation at block level. If a condition is not met, the code after `>>` is not evaluated:

```
{:defined; varname >> {:code; {:code; expensive operation :} :} :}
```

If `varname` is not defined, the nested code blocks are never processed, improving performance.

### Coalesce

Outputs the first non-empty (non-zero length) result from a block list. Nothing is displayed if all blocks are empty:

```
{:coalesce;
    {:;varname1:}
    {:;varname2:}
    {:;varname3:}
    {:code; default value :}
:}
```

This is useful for providing fallback values:
```
{:coalesce;
    {:;user->nickname:}
    {:;user->name:}
    {:code; Anonymous :}
:}
```

### Else Block

The `else` BIF evaluates whether the **previous expression produces an empty block**, not the Boolean expression itself:

```
{:;varname:}{:else; shown if varname is empty :}

{:filled; varname >> content :}{:else; alternative content :}

{:snippet; optional-snippet :}{:else; {:snippet; default-snippet :} :}
```

### Snippets

Snippets are reusable template fragments that can be defined and called multiple times. Calling a snippet that does not exist is not an error; it returns an empty string.

#### Defining Snippets
```
{:snippet; snippet-name >> 
    <div class="card">
        <h2>{:;title:}</h2>
        <p>{:;content:}</p>
    </div>
:}
```

#### Using Snippets
```
{:snippet; snippet-name :}
```

#### Dynamic Snippet Selection
```
{:snippet; option-{:;varname:} :}
{:else; {:snippet; option-default :} :}
```

#### Alias
```
{:snip; name >> content :}    # Same as {:snippet; ... :}
```

### Code Block

The `code` BIF creates a code block for grouping content:

```
{:code; content :}                           # Basic code block
{:code; {:flg; safe :} >> content :}         # Safe mode (no parsing)
{:code; {:flg; noparse :} >> content :}      # No parsing
{:code; {:flg; encode_tags :} >> content :}  # Encode HTML tags
```

### Include

Include external template files:

```
{:include; file.ntpl :}                      # Include template
{:include; {:flg; require :} >> file.ntpl :} # Required include (error if missing)
{:include; {:flg; noparse :} >> file.css :}  # Include without parsing
{:include; {:flg; safe :} >> file.txt :}     # Safe include
{:include; #/relative/path.ntpl :}           # Relative to current file
{:include; header-{:;theme:}.ntpl :}         # Dynamic include (partial)
{:include; {:;varname:} :}                   # Full dynamic - Get error requires allow
```

### Iteration

#### Each Loop
Iterates over arrays/objects:
```
{:each; array key value >>
    <li>{:;key:}: {:;value:}</li>
:}
```

With nested data:
```
{:each; users index user >>
    <div class="user">
        <span>{:;index:}</span>
        <span>{:;user->name:}</span>
        <span>{:;user->email:}</span>
    </div>
:}
```

#### For Loop
Numeric iteration:
```
{:for; var 1..10 >>
    <span>{:;var:}</span>
:}
```

### Parameters

Parameters allow passing values within code blocks:

```
{:code;
    {:param; name >> value :}     # Set parameter
    {:param; name :}              # Get parameter
    
    {:param; title >> Welcome :}
    <h1>{:param; title :}</h1>
:}
```

### Evaluation

The `eval` BIF evaluates content and makes it available via `__eval__`:

```
{:eval; {:;varname:} >>
    Content if not empty: {:;__eval__:}
:}
```

### Comments

Comments are removed from the output:

```
{:* single line comment *:}

{:*
    multi-line
    comment
*:}

{:; varname {:* inline comment *:} :}
```

### Unprintable

The unprintable BIF `{:;:}` outputs an empty string:

```
{:;:}     # Outputs empty string, preserves surrounding whitespace
{:^;:}    # Outputs empty string, eliminates previous whitespace
```

---

## Schema Structure

```json
{
    "config": {
        "comments": "remove",
        "cache_prefix": "neutral-cache",
        "cache_dir": "",
        "cache_on_post": false,
        "cache_on_get": true,
        "cache_on_cookies": true,
        "cache_disable": false,
        "filter_all": false,
        "disable_js": false,
        "debug_expire": 3600,
        "debug_file": ""
    },
    "inherit": {
        "locale": {
            "current": "en",
            "trans": {
                "en": {
                    "Hello": "Hello"
                },
                "es": {
                    "Hello": "Hola"
                }
            }
        },
        "data": {
            "my_var": "value"
        }
    },
    "data": {
        "CONTEXT": {
            "ROUTE": "",
            "HOST": "",
            "GET": {},
            "POST": {},
            "HEADERS": {},
            "FILES": {},
            "COOKIES": {},
            "SESSION": {},
            "ENV": {}
        },
        "my_var": "value",
        "my_obj": {
            "key": "value"
        }
    }
}
```

---

## Safety Features

### Context Variables

User-provided data should be placed in `CONTEXT`:

```json
{
    "data": {
        "CONTEXT": {
            "GET": {},
            "POST": {},
            "COOKIES": {},
            "ENV": {}
        }
    }
}
```

All `CONTEXT` variables are automatically HTML-escaped.

### Security

- Variables are **not evaluated** (template injection safe)
- Complete variable evaluation requires `{:allow; ... :}`
- Partial evaluation allowed: `{:; array->{:;key:} :}`
- Templates cannot access data not in schema
- `CONTEXT` vars auto-escaped. `filter_all: true` escapes everything
- By design, templates do not have access to arbitrary application data; data must be passed in JSON

### File Security

```
# Error - insecure:
{:include; {:;varname:} :}

# Secure - with allow:
{:declare; valid-files >>
    home.ntpl
    login.ntpl
:}
{:include;
    {:allow; valid-files >> {:;varname:} :}
    {:else; error.ntpl :}
:}
```

### Allow/Deny Lists

Note: `declare` must be defined in a file whose name contains the word "snippet". An error will occur if an attempt is made to define it elsewhere.

```
{:declare; allowed >>
    word1
    word2
    *.ntpl
:}
{:allow; allowed >> {:;varname:} :}
```

Wildcards supported:
- `.` - matches any character
- `?` - matches exactly one character
- `*` - matches zero or more characters

Examples:
```
<div>{:allow; templates >> file.txt :}{:else; fails :}</div>
<div>{:allow; languages >> fr :}{:else; fails :}</div>
```

---

## Advanced Features

### Caching

```
{:cache; /300/ >> content :}            # Cache for 300 seconds
{:cache; /300/custom-id/ >> content :}  # Cache with custom ID
{:!cache; content :}                    # Exclude from cache
```

**Cache Example:**
```
{:cache; /120/ >>
    <!DOCTYPE html>
    <html>
    <body>
        <div>{:code; ... :}</div>
        {:!cache; {:date; %H:%M:%S :} :}
    </body>
    </html>
:}
```

If we run it a second later, the cache part will be the same for 120 seconds, while the `!cache` part is always updated.

### Translations

```
{:trans; Hello :}                # Translate text
{:!trans; Hello :}               # Translate or empty
{:locale; translations.json :}   # Load translations
```

Translation schema:
```json
{
    "inherit": {
        "locale": {
            "current": "en",
            "trans": {
                "en": { "Hello": "Hello" },
                "es": { "Hello": "Hola" },
                "de": { "Hello": "Hallo" },
                "fr": { "Hello": "Bonjour" }
            }
        }
    }
}
```

### Object Integration

`obj` allows you to execute scripts in other languages like Python:

```
{:obj; 
    {
        "engine": "Python",
        "file": "script.py",
        "template": "template.ntpl"
    } 
:}
```

---

## Scope & Inheritance

- Definitions are block-scoped, inherited by children, recovered on exit
- `include` has block scope
- `+` modifier promotes definitions to current/parent scope

### Scope Modifier (`+`)

By default, definitions have block scope. `+` extends to current level:

```
{:code;
    {:include; snippet.ntpl :}
    {:snippet; name :}     # Not available outside
:}
{:snippet; name :}         # Not available

{:+code;
    {:include; snippet.ntpl :}
    {:snippet; name :}     # Available outside
:}
{:snippet; name :}         # Still available
```

---

## HTTP Features

### Exit/Redirect

```
{:exit; 404 :}                    # Exit with status
{:!exit; 302 :}                   # Set status, continue
{:exit; 301 >> /url :}            # Redirect
{:redirect; 301 >> /url :}        # HTTP redirect
{:redirect; js:reload:top :}      # JS redirect
```

### Fetch (AJAX)

The system provides basic JavaScript for performing simple fetch requests:

```
{:fetch; |/url|auto| >> <div>loading...</div> :}
{:fetch; |/url|click| >> <button>Load</button> :}
{:fetch; |/url|form| >> ... :}
```

Events: `auto`, `none`, `click`, `visible`, `form`

---

## Configuration Options

```json
{
    "config": {
        "comments": "remove",
        "cache_prefix": "neutral-cache",
        "cache_dir": "",
        "cache_on_post": false,
        "cache_on_get": true,
        "cache_on_cookies": true,
        "cache_disable": false,
        "filter_all": false,
        "disable_js": false,
        "debug_expire": 3600,
        "debug_file": ""
    }
}
```

---

## Quick Reference Table

| BIF | Purpose |
|-----|---------|
| `{:;var:}` | Output variable |
| `{:code; ... :}` | Code block |
| `{:filled; v >> c :}` | Conditional (has content) |
| `{:defined; v >> c :}` | Conditional (is defined) |
| `{:bool; v >> c :}` | Conditional (is true) |
| `{:array; v >> c :}` | Conditional (is array) |
| `{:same; /a/b/ >> c :}` | String comparison |
| `{:contains; /h/n/ >> c :}` | Substring check |
| `{:each; arr k v >> c :}` | Loop through array |
| `{:for; v 1..10 >> c :}` | For loop |
| `{:include; file :}` | Include file |
| `{:snippet; n >> c :}` | Define snippet |
| `{:snippet; n :}` | Play snippet |
| `{:trans; text :}` | Translate |
| `{:cache; /t/ >> c :}` | Cache content |
| `{:!cache; c :}` | Exclude from cache |
| `{:coalesce; ... :}` | First non-empty |
| `{:else; c :}` | Else condition |
| `{:eval; c1 >> c2 :}` | Evaluate and output |
| `{:param; n >> v :}` | Set parameter |
| `{:allow; list >> val :}` | Whitelist check |
| `{:declare; name >> list :}` | Define whitelist |
| `{:exit; code :}` | HTTP status/exit |
| `{:redirect; code >> url :}` | Redirect |
| `{:fetch; \|url\|ev\| >> c :}` | AJAX request |
| `{:join; /arr/sep/ :}` | Join array |
| `{:moveto; tag >> c :}` | Move content to tag |
| `{:date; format :}` | Output date |
| `{:hash; text :}` | MD5 hash |
| `{:rand; 1..10 :}` | Random number |
| `{:sum; /a/b/ :}` | Sum values |
| `{:replace; /f/t/ >> c :}` | String replace |
| `{:count; ... :}` | Count (deprecated) |
| `{:data; file.json :}` | Load local data |
| `{:locale; file.json :}` | Load translations |
| `{:debug; key :}` | Debug output |
| `{:neutral; c :}` | No-parse output |
| `{:obj; config :}` | Execute external script |
| `{:; :}` | Unprintable (empty) |

---

## Security Best Practices

1. **Never trust context data** (GET, POST, COOKIES, ENV)
2. **Use `CONTEXT`** for all user-provided data
3. **Use `{:allow; ... :}`** for dynamic file inclusion
4. **Use `filter_all: true`** for maximum safety
5. **Validate variables** with `declare`/`allow` before evaluation
6. **Remove debug** BIFs before production
7. **Both application and templates** should implement security

### Rust Foundation

Neutral TS template engine is developed in Rust, one of the most secure programming languages. It does not have access to the application's data; it cannot do so because it is designed this way.

### IPC Security Benefits

The IPC architecture provides important security benefits: Sandboxed execution: Templates run in isolated processes. Reduced attack surface: Main application protected from template engine vulnerabilities. Resource control: Memory and CPU limits can be enforced at server level. Crash containment: Template engine failures don't affect the main application. Zero-downtime updates: IPC server can be updated independently without restarting client applications.

### Data Isolation by Design

By design the templates do not have access to arbitrary application data, the data has to be passed to the template in a JSON, then the data that you have not included in the JSON cannot be read by the template.

### Security Summary

| Security Feature | Description |
|------------------|-------------|
| **Sandboxed Execution** | Templates run in isolated processes |
| **Reduced Attack Surface** | Main application protected from template vulnerabilities |
| **Resource Control** | Memory and CPU limits enforceable at server level |
| **Crash Containment** | Template failures don't affect main application |
| **Data Isolation** | Templates only access explicitly passed JSON data |
| **Rust Memory Safety** | Built with one of the most secure programming languages |

---

## Truth Table

| Variable | Value | filled | defined | bool | array |
|----------|-------|--------|---------|------|-------|
| true | true | ✅ | ✅ | ✅ | ❌ |
| false | false | ✅ | ✅ | ❌ | ❌ |
| "hello" | string | ✅ | ✅ | ✅ | ❌ |
| "0" | string | ✅ | ✅ | ❌ | ❌ |
| "1" | string | ✅ | ✅ | ✅ | ❌ |
| " " | spaces | ✅ | ✅ | ✅ | ❌ |
| "" | empty | ❌ | ✅ | ❌ | ❌ |
| null | null | ❌ | ❌ | ❌ | ❌ |
| undef | — | ❌ | ❌ | ❌ | ❌ |
| [] | empty arr | ❌ | ✅ | ❌ | ✅ |
| {...} | object | ✅ | ✅ | ✅ | ✅ |

---

## Usage Examples

### Basic Usage in Rust

```rust
use neutralts::Template;
use serde_json::json;

// Data
let schema = json!({
    "data": {
        "hello": "Hello, World!",
        "site": {
            "name": "My Site"
        }
    }
});

// Create template
// In file.ntpl use {:;hello:} and {:;site->name:} to show data
let template = Template::from_file_value("file.ntpl", schema).unwrap();

// Render template
let content = template.render();

// Status management
let status_code = template.get_status_code();   // e.g.: 200
let status_text = template.get_status_text();   // e.g.: OK
let status_param = template.get_status_param(); // empty if no error
```

### Usage in Python (Native Package)

```python
from neutraltemplate import NeutralTemplate

template = NeutralTemplate("file.ntpl", schema)
contents = template.render()

status_code = template.get_status_code()   # e.g.: 200
status_text = template.get_status_text()   # e.g.: OK
status_param = template.get_status_param()
```

### Usage in Python (IPC Client)

Requires an IPC server and an IPC client:

```python
from NeutralIpcTemplate import NeutralIpcTemplate

template = NeutralIpcTemplate("file.ntpl", schema)
contents = template.render()

status_code = template.get_status_code()   # e.g.: 200
status_text = template.get_status_text()   # e.g.: OK
status_param = template.get_status_param()
```

### Handling Redirections and Errors

If you use the BIF "exit" or "redirect", it is necessary to manage the status codes in the application:

```rust
let template = Template::from_file_value("file.ntpl", schema).unwrap();
let content = template.render();

let status_code = template.get_status_code();   // e.g.: 301
let status_text = template.get_status_text();   // e.g.: Moved Permanently
let status_param = template.get_status_param(); // e.g.: https://example.com

// Act accordingly according to your framework
```

---

## Performance Considerations

The IPC approach introduces performance overhead due to inter-process communication. The impact varies depending on factors... For most web applications, the security and interoperability benefits compensate for the performance overhead.

### Easy Updates

Easy updates: No application recompilation needed - simply update the IPC server for engine improvements.

---

## Conclusions

### Strengths

1. **True Language-Agnostic**: Just like an SQL query returns the same data from any language, a Neutral TS template returns the same HTML from Python, PHP, Rust… with added security isolation.

2. **Security First**: Built in Rust with sandboxed execution, data isolation, and process containment.

3. **Modularity**: Enables creating reusable plugins and snippet libraries that work across all supported languages.

4. **Flexible Caching**: Sophisticated modular caching with `!cache` blocks for dynamic content.

5. **Built-in Localization**: Native support for multi-language applications through JSON-based translations.

6. **Mature Design**: Originally developed in PHP and refined over years before being ported to Rust.

### Ideal Use Cases

- Multi-language backend environments requiring consistent templating
- Applications requiring high security through process isolation
- Teams needing to share templates across different technology stacks
- Projects requiring built-in internationalization support
- PWA/Web APP development with modern frameworks like Actix Web

### Trade-offs

- IPC introduces some performance overhead (acceptable for most web applications)
- Unique syntax requires learning curve for developers familiar with other template engines
- Requires IPC server setup for non-Rust languages

### Final Assessment

Neutral TS represents an innovative approach to web templating by prioritizing language independence and security. Its database-like client-server architecture enables true template portability across programming languages while maintaining consistent output. The Rust foundation provides memory safety and performance, while the IPC architecture offers sandboxed execution that protects host applications from template engine vulnerabilities. For teams working in polyglot environments or requiring high security isolation, Neutral TS offers a compelling solution.

---

## Links and Resources

### Official Documentation
- [Main Documentation](https://franbarinstance.github.io/neutralts-docs/docs/neutralts/)
- [Syntax Documentation](https://franbarinstance.github.io/neutralts-docs/docs/neutralts/doc/)
- [Rust API Documentation](https://docs.rs/neutralts/latest/neutralts/)

### Repositories
- [GitHub Repository](https://github.com/FranBarInstance/neutralts)
- [Crates.io](https://crates.io/crates/neutralts)

### Downloads
- [IPC Server](https://github.com/FranBarInstance/neutral-ipc/releases) - Available from the repository releases
- [IPC Clients](https://github.com/FranBarInstance/neutral-ipc/tree/master/clients) - Language-specific clients available at the repository
- [Examples](https://github.com/FranBarInstance/neutralts-docs/tree/master/examples) - Rust, Python, PHP, Node.js, and Go examples