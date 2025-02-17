# \[spec-002\] Static Plugin Design

This specification describes a methodology for extending applications through "static plugins." Unlike runtime plugins, static plugins are composed at the source-code level through class composition.

By composing source code at development time, static plugins retain static analysis benefits, provide explicit dependencies, and eliminate the overhead typical of runtime plugins.

## **Background**

Static plugins provide several key advantages compared to dynamic plugins:

- **No Magic Behavior** – All code dependencies are clearly visible in the source, with no hidden or magical runtime behavior.
- **Eases Debugging** – Straightforward execution without dynamic events.
- **Full Static Analysis** – All code remains visible during development, enabling comprehensive static analysis including linting and type checking.
- **Zero Runtime Overhead** – No event-based overhead or dynamic loading costs.

### **Core Elements**

Within **Static Plugin Design (SPD)**, two core elements exist:

- **Application:** Target software accepting new features
- **Static Plugins:** Self-contained modules adding functionality

## Understanding the Partial Class Pattern (PCP)

Static Plugin Design builds on the [Partial Class Pattern](../spec-001--partial-class-pattern/README.md), which splits class definitions across multiple files.

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
class WithAge extends UserBase {
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
- **Dependencies:** Shape inheritance order - higher-level partials override lower ones, with base class at bottom

For complete details, see [[spec-001] Partial Class Pattern](../spec-001--partial-class-pattern/README.md).

## Static Plugins

A Static Plugin is a modular unit that adds a specific feature to an application. In essence, it consists of a collection of PCP classes.

### File Structure

A Static Plugin follows this structure:

```text
📁 <plugin name>/
├── 📁 base/         # New classes provided by the plugin to the application
├── 📁 mixins/       # Partials extending existing classes
└── 📁 external/     # References to classes defined by other plugins
```

Plugin folders use the "***With***" prefix followed by a feature name. Follow your language's naming standards, defaulting to snake case (e.g., `with_dob`).

### Base Classes

Base classes are new classes that a plugin provides to the application. Unlike PCP's empty marker classes, SPD base classes implement essential features.

```text
📁 <plugin>/
└── 📁 base/
    ├── <class 1>.lang
    ├── <class 1>_interface.lang
    ├── ...
    ├── <class N>.lang
    └── <class N>_interface.lang
```

**Example**

Consider a plugin `with_users` that provides a `User` class:

```text
📁 with_users/
└── 📁 base/
    ├── user.lang
    └── user_interface.lang
```

The base class should contain the most fundamental features. In this case, we include the "*name*" functionality in the base *User* class. Here's the *User* interface:

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

**Notes**

- **Import Convention:** Following PCP's convention, when importing a self-interface, always use the alias `ImplementsInterface`.
- **Empty Base Classes:** While base classes typically include core functionality, you can create empty base classes that plugins will fully extend if needed.

### Mixins

Mixins extend the functionality of an existing base class, functioning as PCP's partials.

```text
📁 <plugin name>/
└── 📁 mixins/
    ├── <class 1>_mixin.lang
    ├── <class 1>_interface_mixin.lang
    ├── ...
    ├── <class N>_mixin.lang
    └── <class N>_interface_mixin.lang
```

Each mixin's name ends with `Mixin` (e.g., `UserMixin`, `UserInterfaceMixin`).

**Mutual Exclusivity Rule**

A plugin can contain multiple base classes and mixins, but it cannot use mixins to extend its own base classes. Simply put: **One plugin can't both define and extend the same class.**

**Example**

Here's a `with_dob` plugin that adds date of birth to the "*User*" class:

```text
📁 with_dob/
└── 📁 mixins/
    ├── user_mixin.lang
    └── user_interface_mixin.lang
```

Rather than importing directly from other plugins, mixin interfaces use an "external" version that acts as an intermediary. This creates a clean abstraction layer between plugins, which we'll explore in the next section.

```python
# file: ./with_dob/mixins/user_interface_mixin.lang

# external class, i.e. from another plugin
"../external/user_interface" as UserInterface;

# interface with new features for the class
interface UserMixinInterface
    extends UserInterface
{
    void set_dob(dob: Date);
    Date get_dob();
}
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

### External Classes

Instead of importing classes directly from other plugins, which becomes complex with multiple extensions, plugins use "external classes" in the `external/` folder as intermediaries. These external classes serve as a bridge, used only when base classes and mixins need to reference classes from other plugins.

```text
📁 <plugin name>/
└── 📁 external/
    ├── <class 1>_interface.lang
    ├── ...
    └── <class N>_interface.lang
```

External classes match their base class names (e.g. `user_interface`, `book_interface`).

**Focus on Interfaces**

Plugins generally import only interfaces, not concrete implementations, from other plugins. While external concrete classes exist, they're uncommon and will be covered later.

**Import Convention**

When importing classes into external classes, you must follow these naming conventions:

- **Base Class:** `<class>Base` (e.g., "*UserBase*")
- **Base Interface:** `<class>BaseInterface` (e.g., "*UserBaseInterface*")
- **Mixin Class:** `<plugin name>` (e.g., "*WithAge*")
- **Mixin Interface:** `<plugin name>Interface` (e.g., "*WithAgeInterface*")

**Example**

Earlier, we saw that the `with_dob` plugin's mixin interface extends the base interface from `with_users` through an external class interface. Here's how to implement it:

```text
📁 with_dob/
├── 📁 external/
│   └── user_interface.lang
└── ...
```

External interfaces follow the same implementation pattern as PCP's composed interfaces:

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

To use external classes, just import them:

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

### Mixin as External Class

When importing a class from another plugin, use the local mixin instead of its external class if one is available. This ensures access to the most recent mixin-added features.

**Example**

Consider a plugin that adds user role functionality to the application. This plugin introduces a new "Role" class and enhances the existing "*User*" class:

```python
# file ./with_roles/mixins/user_interface_mixin.lang

# self external
"../external/user_interface" as UserInterface;

interface UserInterfaceMixin
    extends UserInterface
{
    str get_role_name();
}
```

When referencing "*User*" from another class in the same plugin, use the local mixin version instead of the external class. This ensures access to all the latest plugin-added functionality:

```python
# file: ./with_roles/base/role_interface.lang

# using the local mixin instead of the external version of the class
"../mixins/user_interface_mixin" as UserInterface;

interface RoleInterface
{
    UserInterface[] get_users();
}
```

**Notes**

- **Import Convention:** When using a mixin interface as an external class, name it `<class>Interface` (e.g. "*UserInterface*") to match regular external class naming.

### Plugin Dependencies

Plugin dependencies are managed through inheritance order in composed classes, following the same approach as PCP dependencies.

**Example**

Let's see how to build a plugin that depends on multiple other plugins. The `with_age` plugin adds age calculation functionality to the *User* class. It has two dependencies:

- `with_users`: provides the base "*User*" class we'll extend
- `with_dob`: provides date of birth functionality for the "*User*" class, which we need to calculate age

**External Interface:** First, we compose the external interface for the *User* class that our mixins will use:

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

**Mixin Interface:** Now we can implement the age calculation functionality in our mixin:

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

**File Structure:** Here's the complete file structure for the plugin:

```text
📁 with_age/
├── 📁 external/
│   └── user_interface.lang
│
└── 📁 mixins/
    ├── user_mixin.lang
    └── user_interface_mixin.lang
```

## SPD Applications

An SPD Application is a collection of static plugins that combines mixins and base classes into composed classes. These composed classes form the application's foundation.

### File Structure

A SPD application has this structure:

```text
📁 <application name>/
├── 📁 plugins/    # Static plugins
├── 📁 final/      # Composed classes from plugins' mixins and bases
└── 📁 support/    # Shared utilities (optional)
```

The structure consists of:

- **`plugins/`**: Contains all static plugins
- **`final/`**: Holds the fully composed classes that combine base classes with their mixins
- **`support/`**: Contains shared utilities and helper functions used across plugins (optional)

Name your application folder following your programming language or framework's naming convention (e.g., "*mybooks*", "*my_books*", "*MyBooks*").

### The `plugins/` Folder

Store all static plugins in the `plugins/` directory:

```text
📁 myapp/
├── 📁 plugins/
│   ├── 📁 with_users/
│   ├── 📁 with_dob/
│   └── 📁 with_age/
└── ...
```

**Or more generally:**

```text
📁 <application>/
├── 📁 plugins/
│   ├── 📁 <plugin 1>
│   ├── ...
│   └── 📁 <plugin N>
└── ...
```

### The `final/` Folder

In an SPD application, composed classes are called "**Final Classes**". Each final class and its interface version is stored in its own file within the `final/` directory:

```text
📁 myapp/
├── 📁 final/
│   ├── user.lang
│   └── user_interface.lang
└── ...
```

**Or more generally:**

```text
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

**Example**

Here's an example of a "*UserInterface*":

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

```text
📁 myapp/
├── 📁 support/
│   ├── 📁 lib_netclient/
│   ├── 📁 lib_md5/
│   └── helpers_datetime.lang
└── ...
```

**Or more generally:**

```text
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

# self interface
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

```text
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

```text
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

```text
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
