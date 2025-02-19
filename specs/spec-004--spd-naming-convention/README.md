# \[spec-004\] SPD Naming Convention

This specification outlines a standard naming convention for Static Plugin Design (SPD) plugin folders to ensure consistent organization.

## Plugin Types

SPD Application plugins are divided into two categories:

- **Base Plugin:** Provides core functionality for the application. Every SPD Application requires exactly one base plugin, which operates independently and serves as a foundation for all other plugins.
- **Feature Plugins:** These plugins add specific functionality to the application. Each feature plugin requires the base plugin and may rely on other feature plugins.

**Example**

Consider an application "*myapp*" with this manifest:

```toml
name = "myapp"

plugins = [
    "with_users",  # lowest precedence
    "with_dob",
    "with_age"     # highest precedence
]
```

Plugins are ordered from lowest precedence (basic features) to highest precedence.

In this example, `with_users` is the **base plugin**, while `with_dob` and `with_age` are **feature plugins**.

## Base Plugin Name

According to this specification, the base plugin must be named `with_base` to clearly distinguish it from feature plugins.

**Example**

```toml
name = "myapp"

plugins = [
    "with_base",  # <--- base plugin
    "with_dob",
    "with_age"
]
```

## Plugin Folder Names

This specification establishes a numbering system for plugin folders. A number is prepended to each plugin name.

Examples:

- `_01_with_base` - Core application functionality
- `_20_with_dob` - Date of birth feature
- `_30_with_age` - Age calculation feature

**Or more generally:**

```
_<number>_<plugin name>
```

### Numbering System

The numbering prefix appears only in folder names and is not part of the plugin name itself. Plugins are numbered sequentially from lowest to highest precedence, matching their order in the application's manifest.

The numbering system follows these rules:

- Two digits minimum (01-99)
- Expand to three digits when exceeding 99
- Numbers must be sequential but can have gaps

This system provides programmers with a clear visual organization of the precedence order.

**Example**

```
📁 myapp/
├── 📁 plugins/
│   ├── 📁 _01_with_base/
│   ├── 📁 _10_with_dob/
│   └── 📁 _20_with_age/
└── ...
```

### Case Convention

While this specification establishes a naming pattern, it does not specify a case convention for the plugin name (e.g., camel case, snake case, pascal case). Use the case convention specified by your language and/or framework (e.g. “*_01_WithBase*”, “*_01_with_base*”).

## References

- [spec-002](https://github.com/thecodecrate/guidelines/blob/main/specs/spec-002--static-plugin-design/README.md) - Static-Plugin Design (SPD)
- [spec-003](https://github.com/thecodecrate/guidelines/blob/main/specs/spec-003--spd-manifest-files/README.md) - SPD Manifest Files
