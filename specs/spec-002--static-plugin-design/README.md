# \[spec-002\] Static-Plugin Design

This specification outlines a methodology for extending applications using “static plugins”.

Static plugins extend applications through class composition in source code, rather than dynamic runtime plugins. This preserves static analysis, prevents side effects, and avoids runtime issues while keeping the benefits of plugin-based extensibility.

## **Background**

Static plugins offer several key advantages over dynamic plugins:

- **No Magic Behavior** – Dependencies are explicitly visible in the source code, eliminating hidden runtime behavior.
- **Eases Debugging** – Code execution follows a clear, predictable path without dynamic events.
- **Full Static Analysis** – All code is available during development, enabling comprehensive linting and type checking.
- **Zero Runtime Overhead** – Eliminates the performance costs of event handling and dynamic loading.

### **Core Elements**

Within **Static Plugin Design (SPD)**, two core elements exist:

- **Application:** Target software accepting new features
- **Static Plugins:** Self-contained modules adding functionality

## Understanding the Partial Class Pattern (PCP)

Static Plugin Design builds on the [Partial Class Pattern](https://github.com/thecodecrate/guidelines/blob/main/specs/spec-001--partial-class-pattern/README.md), which splits class definitions across multiple files.

### Example: Decomposing a Monolithic Class

Let's examine a typical monolithic `User` class that manages name, date of birth (DOB), and age calculations:

```python
class User {
    # Name management
    void set_name(name: str) {}
    str get_name() {}

    # Date of birth (DOB) management
    void set_dob(dob: Date) {}
    Date get_dob() {}

    # Age calculation
    int calculate_age() {}
}
```

### Applying PCP

PCP divides this class into three distinct components:

1. **Base Class** (`UserBase`) — A minimal foundation class
2. **Partial Classes** — Separate implementations for each feature
3. **Composed Class** — The final class that combines all components

Here's how we restructure the code:

```python
# Base class - foundation
class UserBase {}

# Name management partial
class WithName extends UserBase {
    void set_name(name: str) {}
    str get_name() {}
}

# DOB management partial
class WithDob extends UserBase {
    void set_dob(dob: Date) {}
    Date get_dob() {}
}

# Age calculation partial
class WithAge extends WithDob, UserBase {
    int calculate_age() {}
}

# Final composed class
class User
    extends WithAge,    # Highest precedence
            WithDob,
            WithName,
            UserBase    # Lowest precedence
{}
```

### Dependencies

Partials can have dependencies. For example, age calculation needs date of birth functionality.

When A depends on B, list A before B in the inheritance chain for proper method overriding.

**Example**

```python
class User
    extends WithAge,    # Depends on WithDob
            WithDob,    # Provides DOB methods
            UserBase
{}
```

### PCP Key Concepts

- **Components:** Consists of partials, base and composed classes
- **Interfaces:** Each class needs an interface (omitted for brevity)
- **Dependencies:** Set inheritance order - higher-level partials override lower ones, with base class at bottom

For complete details, see [[spec-001] Partial Class Pattern](https://github.com/thecodecrate/guidelines/blob/main/specs/spec-001--partial-class-pattern/README.md).

## Static Plugins

A Static Plugin is a modular unit that adds a specific feature to an application. In essence, it consists of a collection of PCP classes.

### Key Components

- **Base Classes:** New classes provided by the plugin to the application
- **Mixins:** Partials extending existing classes
- **External Classes:** References to classes defined by other plugins

### File Structure

A Static Plugin follows this structure:

```
📁 <plugin name>/
├── 📁 base/
├── 📁 mixins/
└── 📁 external/
```

**Naming Convention:** Plugin folders use the "***With***" prefix followed by a feature name. Follow your language's naming standards, defaulting to snake case (e.g., `with_dob`).

```python
📁 with_dob/
└── ...
```

### Base Classes

Base classes are new classes that a plugin provides to the application. Unlike PCP's empty marker classes, SPD base classes implement essential features.

```
📁 <plugin>/
└── 📁 base/
    ├── <class 1>.lang
    ├── <class 1>_interface.lang
    ├── ...
    ├── <class N>.lang
    └── <class N>_interface.lang
```

**Implementation**

Consider a plugin `with_users` that provides a `User` class:

```
📁 with_users/
└── 📁 base/
    ├── user.lang
    └── user_interface.lang
```

**Base Interface:** The base class should contain the most fundamental features. In this case, we include the "*name*" functionality in the base *User* class. Here's the *User* interface:

```python
# file: ./with_users/base/user_interface.lang

# base interface
interface UserInterface
{
    void set_name(name: str);
    str get_name();
}
```

**Base Concrete:** Here's the concrete *User* class:

```python
# file: ./with_users/base/user.lang

# self-interface
"./user_interface" as ImplementsInterface;

# base concrete
class User
    implements ImplementsInterface
{
    void set_name(name: str) {}
    str get_name() {}
}
```

**Notes**

- **Naming Convention:** Base classes use their class name without additional prefixes or suffixes (e.g. `User`, `UserInterface`).
- **Import Convention:** Following PCP's convention, when importing a self-interface, always use the alias `ImplementsInterface`.
- **Minimal Functionality:** The base class should implement only essential, foundational features.
- **Optional Empty Base Classes:** Although base classes usually contain core functionality, you may create empty base classes for plugins to extend later when appropriate.

### Mixins

Mixins extend the functionality of an existing base class, functioning as PCP's partials.

```
📁 <plugin name>/
└── 📁 mixins/
    ├── <class 1>_mixin.lang
    ├── <class 1>_interface_mixin.lang
    ├── ...
    ├── <class N>_mixin.lang
    └── <class N>_interface_mixin.lang
```

**Implementation**

Consider a `with_dob` plugin that adds date of birth to the *User* class:

```
📁 with_dob/
└── 📁 mixins/
    ├── user_mixin.lang
    └── user_interface_mixin.lang
```

**Mixin Interface:** Rather than importing directly from other plugins, mixin interfaces use an "external" version that acts as an intermediary. This creates a clean abstraction layer between plugins, which we'll explore in the next section.

```python
# file: ./with_dob/mixins/user_interface_mixin.lang

# external class, i.e. from another plugin
"../external/user_interface" as UserInterface;

# mixin interface
interface UserMixinInterface
    extends UserInterface
{
    void set_dob(dob: Date);
    Date get_dob();
}
```

**Mixin Concrete:** The concrete implementation of a mixin follows the same pattern as a PCP partial:

```python
# file: ./with_dob/mixins/user_mixin.lang

# self-interface
"./user_interface_mixin" as ImplementsInterface;

# mixin concrete
interface UserMixin
    implements ImplementsInterface
{
    void set_dob(dob: Date) {}
    Date get_dob() {}
}
```

**Notes**

- **Naming Convention:** Each mixin's name ends with `Mixin` (e.g., `UserMixin`, `UserInterfaceMixin`).
- **No direct plugin access:** Mixins must never directly access classes from other plugins. Instead, they must use "external classes" as intermediaries (detailed in the next section).
- **Mutual Exclusivity Rule:** A plugin can contain multiple base classes and mixins, but it cannot use mixins to extend its own base classes. Simply put: **One plugin can't both define and extend the same class.**

### External Classes

Instead of importing classes directly from other plugins, which becomes complex with multiple extensions, plugins use "**external classes**" in the `external/` folder as intermediaries. These external classes serve as a bridge, used only when base classes and mixins need to reference classes from other plugins.

```
📁 <plugin name>/
└── 📁 external/
    ├── <class 1>_interface.lang
    ├── ...
    └── <class N>_interface.lang
```

**Implementation**

Earlier, we saw that the `with_dob` plugin's mixin interface extends the base interface from `with_users` through an external interface. Here's how to implement it:

```
📁 with_dob/
├── 📁 external/
│   └── user_interface.lang
└── ...
```

**External Interface:** External interfaces follow the same implementation pattern as PCP's composed interfaces:

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

**Usage:** To use external classes, just import them.

```python
# file: ./with_dob/mixins/user_interface_mixin.lang

# self-external
"../external/user_interface" as UserInterface;

# mixin for the User class
interface UserMixinInterface
    extends UserInterface
{
    void set_dob(dob: Date);
    Date get_dob();
}
```

**Notes**

- **Naming Convention:** External classes match their base class names (e.g. `user_interface`, `book_interface`).
- **Focus on Interfaces:** Plugins typically import only interfaces from other plugins, not concrete implementations. External concrete classes exist but are rare and will be discussed later.
- **Import Convention:** When importing classes into external classes, use these naming patterns:
  - **Base Class:** `<class>Base` (e.g., "*UserBase*")
  - **Base Interface:** `<class>BaseInterface` (e.g., "*UserBaseInterface*")
  - **Mixin Class:** `<plugin name>` (e.g., "*WithAge*")
  - **Mixin Interface:** `<plugin name>Interface` (e.g., "*WithAgeInterface*")

### Mixin-External Classes

While external classes provide connections to other plugins, they don't include features from the current plugin. Since mixins extend external classes while adding new features, they can serve as more capable replacements for their parent external classes.

For this reason, when accessing features within the same plugin, use local mixins instead of external classes. This approach ensures you have access to all the latest features added by mixins.

These mixins, when used this way, are called "**mixin-external classes**".

**Implementation**

Let's explore a `with_roles` plugin that handles user roles. It consists of two key components:

- *User* mixin: Adds role-related features to the *User* class
- *Role* base: Defines a new class for the application

**User Mixin:** Extends the *User* class with role management capabilities.

```python
interface UserInterfaceMixin
    extends UserInterface
{
    # new method
    str get_role_name();
}
```

**Role Base:** Requires the *User* interface for implementation.

```python
interface RoleInterface
{
    # external reference "UserInterface"
    UserInterface[] get_users();
}
```

**Using Mixin-External:** We use the mixin version of the *User* interface instead of the external interface directly. This approach is more efficient because the local mixin inherits from the external class while providing additional role functionality:

```python
# INCORRECT: using external interface
"../external/user_interface" as UserInterface;

# CORRECT: using mixin-external interface
"../mixins/user_interface_mixin" as UserInterface;

interface RoleInterface
{
    UserInterface[] get_users();
}
```

**Notes**

- **Compatibility:** Mixins can safely replace their parent external class in imports since they inherit from the external class, ensuring full compatibility.
- **No empty mixins:** Use the mixin version only when it adds functionality. Don't create empty mixins solely to wrap external classes.

### Plugin Dependencies

Plugin dependencies are managed through inheritance order in external classes, following the same approach as PCP dependencies.

**Implementation**

Let's see a plugin that depends on multiple other plugins. The `with_age` plugin adds age calculation functionality to the *User* class. It has two dependencies:

- `with_users`: provides the base "*User*" class we'll extend
- `with_dob`: provides date of birth functionality for the "*User*" class, which we need to calculate age

**External Interface:** This is where we define dependencies in the code. A plugin with no external interfaces has no dependencies:

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

**Mixin Interface:** Now we implement the age calculation functionality in our mixin:

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

**Mixin Implementation:** And the concrete implementation:

```python
# file: ./with_age/mixins/user_mixin.lang

# self-interface
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

**Notes**

- **Externals vs Dependencies:** If a plugin has no external classes, it means it doesn't access any classes from other plugins and therefore has no dependencies. Conversely, if a plugin has external classes, it means it accesses other plugins and thus depends on them.
- **Where is Defined:** The dependency relationship is defined in the external class through its inheritance list (the `extends` clause), following the PCP pattern.
- **PCP Rules:** Plugin dependencies follow the same rules as PCP dependencies—higher-level partials override lower ones, with the base class at the bottom.

### Importing Constraints

Each plugin component follows specific rules for imports, implementations, and extensions. Here's a reference list:

- **Base Interface**
  - **Can Implement:** Nothing (it's an interface)
  - **Can Extend:**
    - Nothing, or
    - 3rd-party interfaces only
  - **Can Use:**
    - Local mixin-external interfaces
    - Local external interfaces
    - Local base interfaces
    - 3rd-party interfaces
- **Base Concrete**
  - **Must Implement:**
    - Self-interface only
  - **Can Extend:**
    - Nothing, or
    - 3rd-party concrete classes only
  - **Can Use:**
    - Local mixin-external classes and interfaces
    - Local external classes and interfaces
    - Local base classes and interfaces
    - 3rd-party classes and interfaces
- **External Interface**
  - **Can Implement:** Nothing (it's an interface)
  - **Must Extend:**
    - Base interface from external dependency (required)
    - Mixin interfaces from external dependencies (optional)
  - **Can Use:** Nothing (should contain no code)
- **External Concrete**
  - **Must Implement:**
    - Self-interface only
  - **Must Extend:**
    - Base concrete from external dependency (required)
    - Mixin concretes from external dependencies (optional)
  - **Can Use:** Nothing (should contain no code)
- **Mixin Interface**
  - **Can Implement:** Nothing (it's an interface)
  - **Must Extend:**
    - Self-external interface
  - **Can Also Extend:**
    - 3rd-party interfaces
  - **Can Use:**
    - Local mixin-external interfaces
    - Local external interfaces
    - Local base interfaces
    - 3rd-party interfaces
- **Mixin Concrete**
  - **Must Implement:**
    - Self-interface only
  - **Can Extend:**
    - Nothing, or
    - 3rd-party concrete classes
  - **Can Use:**
    - Local mixin-external classes and interfaces
    - Local external classes and interfaces
    - Local base classes and interfaces
    - 3rd-party classes and interfaces

## SPD Applications

An SPD Application combines static plugins, their mixins, and base classes to create composed classes that serve as the application's foundation.

### Key Components

- **Plugins:** A collection of static plugins that form the application.
- **Final Classes:** Composed classes that combine mixins and base classes from plugins. These are the application's core building blocks.
- **Support:** Shared utilities and helper functions used across plugins (optional).

### File Structure

A SPD application has this structure:

```
📁 <application name>/
├── 📁 plugins/    # Static plugins
├── 📁 final/      # Composed classes from plugins' mixins and bases
└── 📁 support/    # Shared utilities (optional)
```

**Naming Convention:** Name your application folder following your programming language or framework's naming convention (e.g., "*mybooks*", "*my_books*", "*MyBooks*").

### The `plugins/` Folder

Store all static plugins in the `plugins/` directory:

```
📁 myapp/
├── 📁 plugins/
│   ├── 📁 with_users/
│   ├── 📁 with_dob/
│   └── 📁 with_age/
└── ...
```

**Or more generally:**

```
📁 <application>/
├── 📁 plugins/
│   ├── 📁 <plugin 1>
│   ├── ...
│   └── 📁 <plugin N>
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

Final classes are named to match the class they implement (e.g., "*User*") and follow the same implementation pattern used by external classes.

**Implementation**

**Final Interface:** Here's an example of a "*UserInterface*":

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

**Final Concrete:** And here's the final "*User*" class:

```python
# file: ./final/user.lang

# self-interface
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

## Special Cases

### Concrete External Classes

External classes are typically interfaces. While concrete implementations of external classes are rarely needed, they are permitted when necessary. When implementing a concrete external class, simply follow the standard PCP composed class pattern:

```python
# file: ./with_age/external/user.lang

# self-interface
"./user_interface" as ImplementsInterface;

# base
"../../with_users/base/user" as UserBase;

# mixins
"../../with_dob/mixins/user" as WithDob;

class User
    extends WithDob,
            UserBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

### Implementing a Plugin as an Application

Anything can be implemented as an SPD Application, including plugins themselves. When using this approach, the sub-application's final classes are mapped to base classes. This sub-application is called "core" since it represents the plugin's core implementation.

**File Structure**

A plugin implemented as an SPD Application uses this structure:

```
📁 <plugin name>/
├── 📁 base/ -> "./core/final/"    # symlink
│
└── 📁 core/                       # SPD application
    ├── 📁 plugins/
    ├── 📁 final/
    └── 📁 support/
```

The key components are:

- `core/`: The directory that contains the complete SPD application
- `base/`: A symbolic link pointing to the final classes in core

**Alternative without Symlinks**

If your platform doesn't support symlinks, or you prefer not to use them, you can manually create wrapper classes in the "*base*" folder that act as aliases to the classes in core's final.

```
📁 <plugin name>/
├── 📁 base/    # contains wrappers for core's final classes
│   └── ...
│
└── 📁 core/    # main SPD application
    ├── 📁 plugins/
    ├── 📁 final/
    └── 📁 support/
```

**Example**

Here's how a Devise-like authentication plugin would be structured:

```
📁 with_authentication/
├── 📁 base/ -> "./core/final/"   # symlink
│
└── 📁 core/
    ├── 📁 plugins/
    │   ├── 📁 with_database_authenticable/
    │   ├── 📁 with_omniauthable/
    │   ├── 📁 with_confirmable/
    │   ├── 📁 with_recoverable/
    │   ├── 📁 with_registerable/
    │   ├── 📁 with_trackable/
    │   ├── 📁 with_timeoutable/
    │   ├── 📁 with_validatable/
    │   └── 📁 with_lockable/
    └── ...
```
