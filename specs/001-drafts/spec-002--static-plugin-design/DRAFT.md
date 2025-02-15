# [spec-002] Static Plugin Design

This specification describes a methodology for extending applications through "static plugins." Unlike runtime plugins, static plugins are composed at the source-code level through class composition.

By composing source code at compile time, static plugins retain static analysis benefits, provide explicit dependencies, and eliminate the overhead typical of runtime plugins.

## Background

Building code with static plugins offers several advantages:

- **Maintains Full Static Analysis** – All code remains visible during development, ensuring compile-time checks.
- **Provides Explicit Dependencies** – Each plugin clearly declares its required plugins and features.
- **Incur Zero Runtime Overhead** – No event-based overhead or dynamic loading costs.
- **Eases Debugging** – Straightforward execution without dynamic events.

Within **Static Plugin Design (SPD)**, two core elements exist:

- **Application** – The target software that accepts additional features.
- **Static Plugins** – Self-contained modules that introduce or extend functionality.

## Understanding the Partial Class Pattern (PCP)

Before diving into Static Plugin Design, we need to understand the Partial Class Pattern (PCP) as described in [[spec-001] Partial Class Pattern](https://www.notion.so/spec-001-Partial-Class-Pattern-17a011720877805c9872f3ab401ad1ab?pvs=21).

### Core Concept

Partial Class Pattern (PCP) is a method for organizing code by splitting a class definition across multiple files. Each file contains a specific portion of the class functionality, and these portions combine to form a complete class.

### Simple Example

Let's start with a monolithic `User` class that has several responsibilities:

```python
class User {
    # name
    void set_name(name: str) {}
    str get_name() {}

    # date of birth
    void set_dob(dob: Date) {}
    Date get_dob() {}

    # age calculation
    int calculate_age() {}
}
```

> Note: The code examples on this document are language-agnostic pseudo-code, borrowing features from Python, C#, and other languages, solely for illustration.
>

Using PCP, we decompose it into:

1. **Base Class (”*UserBase*”)** – an empty marker class for PCP.
2. **Partial Classes (”*WithName*”, “*WithDob*”, “*WithAge*”)** – each partial inherits from `UserBase` and implements a focused feature.
3. **Composed Class (”*User*”)** – inherits from all partials and `UserBase`.

```python
# base
class UserBase {}

# partial 1: name
class WithName extends UserBase {
    void set_name(name: str) {}
    str get_name() {}
}

# partial 2: date of birth
class WithDob extends UserBase {
    void set_dob(dob: Date) {}
    Date get_dob() {}
}

# partial 3: age calculation
class WithAge extends UserBase {
    int calculate_age() {}
}

# composed
class User
    extends WithAge,   # last to override
            WithDob,
            WithName,
            UserBase,  # base
{
}
```

Each piece includes an interface that enforces type safety. The final composition yields a single class with all features.

## Static Plugins

A Static Plugin is a modular unit that adds a specific feature to an application. At its core, Static Plugins consist of classes organized using the Partial Class Pattern.

### File Structure

A Static Plugin uses this file structure:

```
📁 <plugin name>/
├── 📁 base/         # New classes provided by the plugin to the application
├── 📁 mixins/       # Partials extending existing classes
├── 📁 external/     # References to classes defined by other plugins
│
└── _plugin.toml     # Manifest with plugin's settings
```

When naming plugin folders, follow the same convention as PCP's partial naming: prefix "***With***" followed by a feature description. Use your language's standard case for files and folders. If no standard exists, we recommend snake case (e.g., `with_dob`). For example:

```
📁 with_dob/
│  ...
└──
```

### Plugin's Manifest

Each plugin requires a manifest file named `_plugin.toml` in its root directory. While you can store any plugin-related information in this TOML-formatted file, it must include these required fields:

- `name`: A unique string identifier for the plugin within the application. We recommend matching the plugin folder name (e.g., `"with_dob"`).
- `dependencies`: An array of strings listing the plugin identifiers that this plugin depends on (e.g., `["with_users"]`).

**Example**

Plugin with no dependencies:

```toml
# ./with_users/_plugin.toml

name = "with_users"
dependencies = []
```

Plugin with a dependency:

```toml
# ./with_dob/_plugin.toml

name = "with_dob"
dependencies = ["with_users"]
```

Plugin with multiple dependencies:

```toml
# ./with_age/_plugin.toml

name = "with_age"
dependencies = ["with_users", "with_dob"]
```

### Base Classes

Base classes are new classes that a plugin provides to the application. Unlike PCP's empty marker classes, SPD base classes contain core functionality.

```
📁 <plugin>/
├── 📁 base/
│   ├── <class 1>.lang
│   ├── <class 1>_interface.lang
│   ├── ...
│   ├── <class N>.lang
│   └── <class N>_interface.lang
│
└── _plugin.toml
```

**Example**

Consider a plugin `with_users` that provides a `User` class:

```
📁 with_users/
├── 📁 base/
│   ├── user.lang
│   └── user_interface.lang
│
└── _plugin.toml
```

A class's most fundamental feature should be part of its base. In this case, we include the "name" functionality in the base *User* class. Here's the *User* interface:

```python
# file: ./with_users/base/user_interface.lang

# base interface
interface UserInterface
{
    void set_name(name: str);
    str get_name();
}
```

And here's the concrete *User* class:

```python
# file: ./with_users/base/user.lang

# self interface
"./user_interface" as ImplementsInterface;

# base class' concrete implementation
class User
    implements ImplementsInterface
{
    void set_name(name: str) {}
    str get_name() {}
}
```

**Import convention**

Following PCP's convention, when importing a self-interface, always use the alias `ImplementsInterface`.

**Empty Base Classes**

Including basic functionality in the base class eliminates the need for separate plugins—one for the empty base and another for core features. However, you may sometimes want an empty base class that plugins fully extend, which this spec allows.

### Mixins

Mixins extend the functionality of an existing base class, serving as the equivalent of PCP's partials.

**File Structure**

```
📁 <plugin name>/
├── 📁 mixins/
│   ├── <class 1>_mixin.lang
│   ├── <class 1>_interface_mixin.lang
│   ├── ...
│   ├── <class N>_mixin.lang
│   └── <class N>_interface_mixin.lang
│
└── _plugin.toml
```

For naming convention, append "***Mixin***" to the class name:

- **Mixin Class:** `<class>Mixin` (e.g., “*UserMixin*”, "*user_mixin*")
- **Mixin Interface:** `<class>InterfaceMixin` (e.g., “*UserInterfaceMixin*”, "*user_interface_mixin*")

**Dependencies**

Mixins rely on base classes, which must be defined in separate plugins. A plugin cannot define both a base class and its mixin. The plugin must declare these dependencies in its manifest file.

**Example**

Let's examine a `with_dob` plugin that adds date of birth functionality to the "*User*" class through a mixin:

```
📁 with_dob/
├── 📁 mixins/
│   ├── user_mixin.lang
│   └── user_interface_mixin.lang
│
└── _plugin.toml
```

Since the "*User*" base class is defined in "*with_users*", we declare this dependency:

```toml
# file: ./with_dob/_plugin.toml

name = "with_dob"
dependencies = ["with_users"]
```

The concrete implementation of a mixin follows the same pattern as a PCP partial:

```python
# file: ./with_dob/mixins/user_mixin.lang

# self interface
"./user_interface_mixin" as ImplementsInterface;

# code with new features for the class
interface UserMixin
    implements ImplementsInterface
{
    void set_dob(dob: Date) {}
    Date get_dob() {}
}
```

The interface follows PCP's pattern, but with one key difference: instead of importing classes directly from other plugins, we import them from the `external/` folder:

```python
# file: ./with_dob/mixins/user_interface_mixin.lang

# import external classes
"../external/user_interface" as UserInterface;

# interface with new features for the class
interface UserMixinInterface
    extends UserInterface
{
    void set_dob(dob: Date);
    Date get_dob();
}
```

### External Classes

When a plugin has dependencies (like plugins with mixins), it needs to import classes from those dependencies. Rather than importing these classes directly, which can become complex when multiple plugins extend them, we create local composed versions called "external classes" in the `external/` folder.

**File Structure**

```
📁 <plugin name>/
├── 📁 external/
│   ├── <class 1>_interface.lang
│   ├── ...
│   └── <class N>_interface.lang
│
└── _plugin.toml
```

**External Interfaces and Classes**

Plugins typically only need to import interfaces from other plugins, not their concrete implementations. External concrete classes exist but are uncommon and reserved for special cases.

Since external concrete classes are a specialized topic we'll address later, let's focus on external interfaces for now.

**Example**

In our `with_dob` plugin, the mixin's interface extends the base interface from "*with_users*". We use an external class interface instead of importing directly:

```
📁 with_dob/
├── 📁 external/
│   └── user_interface.lang
│
└── ...
```

External classes follow the same implementation pattern as PCP's composed classes:

```python
# file: ./with_dob/external/user_interface.lang

# base interface
"../../with_users/base/user_interface" as UserBaseInterface;

interface UserInterface
    extends UserBaseInterface
{
    # intentionally empty
}
```

**Importing Base Classes**

When importing a base class in a composed class definition (such as an external class), use:

- `<class>Base` for concrete implementations (e.g., "*UserBase*")
- `<class>BaseInterface` for interfaces (e.g., "*UserBaseInterface*")

This aligns with PCP naming conventions.

```python
# file: ./with_dob/external/user_interface.lang

# base interface
"../../with_users/base/user_interface" as UserBaseInterface;

...
```

**Importing Mixins**

When importing mixins in an external class definition, name them to match their plugin names (e.g., "*WithAge*", "*WithAgeInterface*"). Here's an example:

```python
# file: ./with_isbn/external/book_interface.lang

# base interface
"../../with_books/base/book_interface" as BookBaseInterface;

# mixins
"../../with_author/mixins/book_interface_mixin" as WithAuthorInterface;
"../../with_genre/mixins/book_interface_mixin" as WithGenreInterface;
"../../with_isbn/mixins/book_interface_mixin" as WithIsbnInterface;
"../../with_owner/mixins/book_interface_mixin" as WithOwnerInterface;

interface BookInterface
    extends WithOwnerInterface,    # last mixin to override
            WithIsbnInterface,
            WithGenreInterface,
            WithAuthorInterface,   # first mixin to override
            BookBaseInterface,     # base
{
    # intentionally empty
}
```

**Usage in Mixins**

Mixins should use the external version of a class rather than the original:

```python
# file: ./with_dob/mixins/user_interface_mixin.lang

# parent interface: using the external version
"../external/user_interface" as UserInterface;

# mixin for the User class
interface UserMixinInterface
    extends UserInterface
{
    void set_dob(dob: Date);
    Date get_dob();
}
```

### Example with Multiple Dependencies

Let's see how to build a plugin that depends on multiple other plugins. The `with_age` plugin adds age calculation functionality to the *User* class. It has two dependencies:

- `with_users`: provides the base "*User*" class we'll extend
- `with_dob`: provides date of birth functionality for the "*User*" class, which we need to calculate age

```
📁 with_age/
├── 📁 external/
│   └── user_interface.lang
│
├── 📁 mixins/
│   ├── user_mixin.lang
│   └── user_interface_mixin.lang
│
└── _plugin.toml
```

First, we declare these dependencies in the manifest:

```toml
# file: ./with_age/_plugin.toml

name = "with_age"
dependencies = ["with_users", "with_dob"]
```

Next, we compose the external *User* interface that our mixins will use:

```python
# file: ./with_age/external/user_interface.lang

# base interface
"../../with_users/base/user_interface" as UserBaseInterface;

# mixin interfaces
"../../with_dob/mixins/user_interface_mixin" as WithDobInterface;

interface UserInterface
    extends WithDobInterface,     # last to override
            UserBaseInterface,    # base

{
    # intentionally empty
}
```

Now we can implement the age calculation functionality in our mixin:

```python
# file: ./with_age/mixins/user_interface_mixin.lang

# self external
"../external/user_interface" as UserInterface;

interface UserInterfaceMixin
    extends UserInterface
{
    int get_age();
}
```

And the concrete implementation:

```python
# file: ./with_age/mixins/user_mixin.lang

# self interface
"./user_interface_mixin" as ImplementsInterface;

class UserMixin
    implements ImplementsInterface
{
    int get_age() {
        # Calculate age using get_dob() from with_dob plugin
        dob = this.get_dob();
        return calculate_years_since(dob);
    }
}
```

## SPD Applications

An SPD Application is a collection of static plugins that combines mixins and base classes into composed classes. These composed classes form the application's foundation.

### File Structure

A SPD application has this structure:

```
📁 <application name>/
├── 📁 plugins/    # Static plugins
├── 📁 final/      # Composed classes from plugins' mixins and bases
├── 📁 support/    # Shared utilities (optional)
│
└── _app.toml      # Manifest with application info
```

The structure consists of:

- **`plugins/`**: Contains all static plugins
- **`final/`**: Holds the fully composed classes that combine base classes with their mixins
- **`support/`**: Contains shared utilities and helper functions used across plugins (optional)
- **`_app.toml`**: Contains information about the application

Name your application folder following your programming language or framework's naming convention (e.g., "*mybooks*", "*my_books*", "*MyBooks*").

### Application's Manifest

Every SPD application needs a `_app.toml` manifest file in its root directory. This TOML-formatted file must include:

- `name`: A string identifier for the application, preferably matching the plugin folder name (e.g., `"myapp"`)
- `plugins`: An array of plugin identifiers that your application depends on (e.g., `["with_users", "with_dob", "with_age"]`), matching each plugin's manifest identifier

**Example**

```toml
# myapp/_app.toml

name = "myapp"
plugins = ["with_users", "with_dob", "with_age"]
```

### The `plugins/` Folder

Store all static plugins in the `plugins/` directory:

```
📁 myapp/
├── 📁 plugins/
│   ├── 📁 with_users/
│   ├── 📁 with_dob/
│   └── 📁 with_age/
│
└── ...
```

Or more generally:

```
📁 <application>/
├── 📁 plugins/
│   ├── 📁 <plugin 1>
│   ├── ...
│   └── 📁 <plugin N>
│
└── ...
```

### The `final/` Folder

In an SPD application, composed classes are called "**Final Classes**". Each final class and its interface version is stored in its own file within the `final/` directory:

```
📁 myapp/
├── 📁 final/
│   ├── user.lang
│   └── user_interface.lang
└── ...
```

**Or more generally:**

```
📁 <application>/
├── 📁 final/
│   ├── <class 1>.lang            # composed class 1
│   ├── <class 1>_interface.lang  # composed interface 1
│   ├── ...
│   ├── <class N>.lang            # composed class N
│   └── <class N>_interface.lang  # composed interface N
└── ...
```

Final classes follow standard PCP composition. Here's an example of a "*UserInterface*":

```python
# file: ./final/user_interface.lang

# base
"../plugins/with_users/base/user_interface" as UserBaseInterface;

# mixins
"../plugins/with_dob/mixins/user_interface_mixin" as WithDobInterface;
"../plugins/with_age/mixins/user_interface_mixin" as WithAgeInterface;

interface UserInterface
    extends WithAgeInterface,  # last to override
            WithDobInterface,
            UserBaseInterface
{
    # intentionally empty
}
```

And here's the final "*User*" class:

```python
# file: ./final/user.lang

# self interface
"./user_interface" as ImplementsInterface;

# base
"../plugins/with_users/base/user" as UserBase;

# mixins
"../plugins/with_dob/mixins/user_mixin" as WithDob;
"../plugins/with_age/mixins/user_mixin" as WithAge;

class User
    extends WithAge,  # last to override
            WithDob,
            UserBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

### The `support/` Folder

Store shared utilities and configuration in the `support/` folder:

```
📁 myapp/
├── 📁 support/
│   ├── 📁 lib_netclient/
│   ├── 📁 lib_md5/
│   └── helpers_datetime.lang
└── ...
```

**Or more generally:**

```
📁 <application>/
├── 📁 support/  # shared utilities and configuration
└── ...
```

### Application Usage

Using an SPD application is straightforward—simply use its final classes.

**Example**

```python
# import application's final classes
"myapp/final/user" as User;

user = User();

# use methods from different plugins
user.set_name("Alice");
user.set_dob("1990-05-15");

# "34" (as of 2025)
print(user.get_age());
```
