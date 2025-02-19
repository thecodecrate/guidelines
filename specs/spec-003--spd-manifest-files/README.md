# \[spec-003\] SPD Manifest Files

This specification defines manifest files for [Static-Plugin Design (SPD)](../spec-002--static-plugin-design/README.md). These files provide metadata for SPD applications and their plugins, containing essential information and settings.

## Plugin Manifest

Each plugin should include a `_plugin.toml` manifest file in its root directory:

```text
📁 <plugin>/
├── _plugin.toml     # Plugin configuration
└── ...
```

Required fields in `_plugin.toml`:

- `name`: Unique identifier for the plugin within the application, preferably matching the plugin folder name (e.g., "*with_dob*").
- `dependencies`: Array of plugin identifiers that this plugin depends on (e.g., "*with_users*"). List dependencies from lowest to highest precedence.

**Examples**

```toml
# No dependencies
name = "with_users"
dependencies = []
```

```toml
# Single dependency
name = "with_dob"
dependencies = ["with_users"]
```

```toml
# Multiple dependencies
name = "with_age"
dependencies = [
    "with_users",  # base
    "with_dob"     # required for age calculation
]
```

## Application Manifest

Applications require a `_app.toml` manifest in their root:

```text
📁 <application>/
├── _app.toml      # Application configuration
└── ...
```

Required fields in `_app.toml`:

- `name`: Application identifier (e.g., "myapp")
- `plugins`: Array of enabled plugins in precedence order

**Example**

```toml
name = "myapp"
plugins = [
    "with_users",
    "with_dob",
    "with_age"
]
```
