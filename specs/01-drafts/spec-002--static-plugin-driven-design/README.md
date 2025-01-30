# [spec-002] Static Plugin Design

This specification describes a methodology for extending applications through static plugins. Unlike runtime plugins, static plugins are composed at source-code level through class composition.

## Background

By composing source code at compile time, Static Plugins retains static analysis benefits, promotes explicit dependencies, and avoids the overhead typical of runtime plugins. This method:

- **Maintains Full Static Analysis** – All code is visible at development, guaranteeing compile-time checks.
- **Provides Explicit Dependencies** – Each plugin states its required plugins and features.
- **Incur Zero Runtime Overhead** – No event-based overhead or dynamic loading.
- **Eases Debugging** – Straight-line execution without dynamic events.

Within **Static Plugin Design (SPD)**, two core elements emerge:

- **Application** – The target software awaiting additional features.
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

Using PCP, we decompose it into:

1. **Base Class (UserBase)** – an empty marker class for PCP.
2. **Partial Classes (WithName, WithDob, WithAge)** – each partial inherits from `UserBase` and implements a focused feature.
3. **Composed Class (User)** – inherits from all partials and `UserBase`.

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
            UserBase,
{
}
```

Each piece includes an interface that enforces type safety. The final composition yields a single class with all features.

## Static Plugins

A Static Plugin is a self-contained module that implements a specific feature across multiple classes. Unlike runtime plugins, static plugins are composed at source code level and are tightly integrated with the application at compile time.

### Structure

A Static Plugin follows this structure:

```
📁 <plugin>/
├── 📁 base/         # Base classes for new types
├── 📁 mixins/       # Extensions for existing types
└── 📁 external/     # Composed external dependencies
```

### Base Classes

Unlike PCP's empty marker base classes, SPD base classes include basic functionality. This eliminates the need for separate plugins—one for the empty base and another for basic features.

**Example**

Let's add basic functionality directly into base classes. Typically, a class's most fundamental feature becomes part of its base:

```python
# file: ./with_users/base/user_interface.lang

# Base class with basic functionality ("with_name")
class UserInterface
{
    void set_name(name: str);
  
    str get_name();
}
```

The "User" concrete class:

```python
# file: ./with_users/base/user.lang

# self interface
"./user_interface" as ImplementsInterface;

# Base class with basic functionality ("with_name")
class User
    implements ImplementsInterface
{
    void set_name(name: str) {}
  
    str get_name() {}
}
```

> Note: For brevity, we are not showing, but we do the same to "*Book*" by incorporating "*WithTitle*" to its base class.
>

This approach of including basic functionality in base classes creates a cleaner plugin structure:

```python
📁 plugins/
├── 📁 with_users/  # User base has "name" functionality
├── 📁 with_books/  # Book base has "title" functionality
└── 📁 with_owner/
```

### Mixins

Organizing files by feature (rather than by class) makes it easier to work on functionality that affects multiple classes. For instance, our current "*with_owner*" plugin extends the "*Book*" class with new methods. However, we may want it to also extend the "User" class with book ownership methods. This would require two partials for the feature: one for Book and another for User.

```python
# This plugin affects two classes
📁 with_owner/
├── <partial_for_book>.lang
└── <partial_for_user>.lang
```

Since we can't use the same filename `with_owner.lang` for both partials, SPD introduces a different naming convention for partials inside plugins.

In SPD, "***partials***" are called "***mixins***". This distinction helps avoid confusion, since "partial" is synonymous with "feature," and features in SPD are implemented as plugins. The term "mixin" better describes these classes' true nature.

### Mixin Naming Convention

Mixin classes follow this naming pattern: `<class>_mixin` for classes and `<class>_interface_mixin` for interfaces. For example:

- Class: `UserMixin`
- Interface: `UserInterfaceMixin`
- File: `user_mixin`, `user_interface_mixin`

Adjust case conventions to match your language's standards.

**Example**

Here's how we name the two partials from our previous example:

```python
📁 with_owner/
├── book_mixin.lang  # partial for Book
└── user_mixin.lang  # partial for User
```

Here are all partials renamed as mixins:

```python
# Plugin 1: Adds users to the application
📁 with_users/
└── user_base.lang

# Plugin 2: Adds books to the application
📁 with_books/
└── book_base.lang

# Plugin 3: Users have a name
📁 with_name/
└── user_mixin.lang

# Plugin 4: Books have a title
📁 with_title/
└── book_mixin.lang

# Plugin 5: Books have an owner
📁 with_owner/
├── book_mixin.lang
└── user_mixin.lang
```

The code remains unchanged. Mixins are simply partials with a different name:

```python
# file: ./with_title/book_mixin.lang

# self interface
"./book_interface_mixin" as ImplementsInterface;

class BookMixin
    implements ImplementsInterface
{
    void set_title(title: str) {}

    str get_title() {}
}
```

### External Classes

In SPD, plugins cannot directly import base classes from other plugins since these classes may have been extended. Instead, we generate local composed versions called "**external classes**" and store them in the `external/` folder. Typically, we only need to import interfaces of classes.

**File Structure**

```python
📁 <plugin>/
├── 📁 external/
│   ├── <class1>_interface.lang
│   ├── ...
│   └── <classN>_interface.lang
│
├── 📁 base/
└── 📁 mixins/
```

**Example**

```python
📁 with_owner/
├── 📁 external/
│   ├── book_interface.lang
│   └── user_interface.lang
│
└── 📁 mixins/
    ├── book.lang
    ├── book_interface.lang
    ├── user.lang
    └── user_interface.lang
```

**Implementation Example**

External classes are implemented in the same way as composed classes in PCP:

```python
# file: ./with_owner/external/book_interface.lang

# base interface
"../../with_books/base/book_interface" as BookBaseInterface;

# mixin interfaces (let's say we still have "with_title")
"../../with_title/mixins/book_interface" as WithTitleInterface;

interface BookInterface
    extends WithTitleInterface,
            BookBaseInterface
{
    # intentionally empty
}
```

The “User” external class:

```python
# file: ./with_owner/external/user_interface.lang

# base interface
"../../with_users/base/user_interface" as UserBaseInterface;

# mixin interfaces
# ... this plugin doesn't depends on any other plugin

interface UserInterface
    extends UserBaseInterface
{
    # intentionally empty
}
```

**Using External Classes**

Mixins and base classes in a plugin must use the external classes instead of importing directly classes from other plugins.

**Example**

Let’s add methods for ownership to “*Book*”:

```python
# file: ./with_owner/mixins/book_interface_mixin.lang

# self external
"../external/book_interface" as BookInterface;

# imported classes
"../external/user_interface" as UserInterface;

interface BookInterfaceMixin
    extends BookInterface
{
    void set_owner(user: UserInterface);
  
    UserInterface get_owner();
}
```

The concrete class for “*Book*”:

```python
# file: ./with_owner/mixins/book_mixin.lang

# self interface
"./book_interface_mixin" as ImplementsInterface;

# imported classes
"../external/user_interface" as UserInterface;

class BookMixin implements ImplementsInterface
{
    void set_owner(user: UserInterface) {}
  
    UserInterface get_owner() {}
}
```

We also add methods to “*User*”:

```python
# file: ./with_owner/mixins/user_interface_mixin.lang

# self external
"../external/user_interface" as UserInterface;

# imported classes
"../external/book_interface" as BookInterface;

interface UserInterfaceMixin
    extends UserInterface
{
    BookInterface[] list_books();
}
```

The concrete class for “*User*”:

```python
# file: ./with_owner/mixins/user_mixin.lang

# self interface
"./user_interface_mixin" as ImplementsInterface;

# imported classes
"../external/book_interface" as BookInterface;

class UserMixin implements ImplementsInterface
{
    BookInterface[] list_books() {}
}
```

## SPD Applications

An SPD Application is a program built by composing multiple Static Plugins. Let's see how this composition evolves:

### Initial Approach: Simple Directory

The simplest way to organize an application with Static Plugins would be placing all files in a single directory:

```bash
📁 mybooks_app/
│   # user classes
├── user.lang          # Composed class
├── user_base.lang     # Base class
│
│   # book classes
├── book.lang          # Composed class
├── book_base.lang     # Base class
│
│   # partial: user has a name
├── with_name.lang
│
│   # partial: book has a title
├── with_title.lang
│
│   # partial: book has an owner
└── with_owner.lang
```

### Structured Approach: The SPD Way

While the simple approach works, SPD prescribes a more organized structure using three key directories:

```
📁 <application>/
├── 📁 final/      # Composed application classes
├── 📁 plugins/    # Static plugin modules
└── 📁 support/    # Shared utilities (optional)
```

Below is an overview of how each folder fits into the bigger picture, followed by more details.

### The `final/` Folder

In SPD, composed classes are known as **Final Classes**. They reside in `final/`, one file per final class:

```
📁 mybooks_app/
├── 📁 final/
│   ├── user.lang
│   └── book.lang
└── ...
```

**Or more generally:**

```
📁 <application>/
├── 📁 final/  # final (composed) classes
└── ...
```

Final classes are just standard composed classes. We will see them in more details later.

### The `support/` Folder

If the application has shared utilities or configuration, SPD places them in a separate `support/` folder:

```
📁 mybooks_app/
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

### The `plugins/` Folder

SPD groups files semantically by feature. Each feature is a "**static plugin**" with its own folder. The folder name reflects the feature's goal, following the same pattern as partials - plugins are partials applied horizontally across multiple classes.

Here's how we might first organize our files into individual static-plugins:

```python
# Plugin 1: Adds users to the application
📁 with_users/
└── user_base.lang

# Plugin 2: Adds books to the application
📁 with_books/
└── book_base.lang

# Plugin 3: Users have a name
📁 with_name/
└── with_name.lang

# Plugin 4: Books have a title
📁 with_title/
└── with_title.lang

# Plugin 5: Books have an owner
📁 with_owner/
└── with_owner.lang
```

All static plugins go into the `plugins/` directory:

```
📁 mybooks_app/
├── 📁 final/
│   ├── user.lang
│   └── book.lang
└── 📁 plugins/
    ├── 📁 with_users/
    ├── 📁 with_books/
    ├── 📁 with_name/
    ├── 📁 with_title/
    └── 📁 with_owner/
```

Or more generally:

```
📁 <application>/
├── 📁 final/   # final classes
└── 📁 plugins/ # static plugins
```

### Final Classes Implementation

Final classes, just as external classes, are composed in the same way as in PCP.

**Example**

Let’s wrap up our application by writing the final classes:

```python
# file: ./final/book_interface.lang

# base interface
"../plugins/with_books/base/book_interface" as BookBaseInterface;

# mixin interfaces (let's say we still have "with_title")
"../plugins/with_title/mixins/book_interface" as WithTitleInterface;
"../plugins/with_owner/mixins/book_interface" as WithOwnerInterface;

interface BookInterface
    extends WithOwnerInterface,  # last to override
            WithTitleInterface,
            BookBaseInterface
{
    # intentionally empty
}
```

The final “*Book*” class:

```python
# file: ./final/book.lang

# self interface
"./book_interface" as ImplementsInterface;

# base
"../plugins/with_books/base/book_interface" as BookBase;

# mixins (let's say we still have "with_title")
"../plugins/with_title/mixins/book" as WithTitle;
"../plugins/with_owner/mixins/book" as WithOwner;

class Book
    extends WithOwner,  # last to override
            WithTitle,
            BookBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

The final “*User*” interface:

```python
# file: ./final/user_interface.lang

# base interface
"../plugins/with_users/base/user_interface" as UserBaseInterface;

# mixin interfaces
"../plugins/with_owner/mixins/user_interface" as WithOwnerInterface;

interface UserInterface
    extends WithOwnerInterface,  # last to override
            UserBaseInterface
{
    # intentionally empty
}
```

The final “*User*” class:

```python
# file: ./final/user.lang

# self interface
"./user_interface" as ImplementsInterface;

# base
"../plugins/with_users/base/user_interface" as UserBase;

# mixins
"../plugins/with_owner/mixins/user" as WithOwner;

class User
    extends WithOwner,  # last to override
            UserBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

### Application Usage

To use the application, just use the final classes.

**Example**

```python
# import application's final classes
"./final/book" as Book;
"./final/user" as User;

# Create a user
user = User();
user.set_name("Alice");

# Create a book and set their owner
book = Book();
book.set_title("The Great Gatsby");
book.set_owner(user);

# List user's books
print(user.list_books())  # prints: ["The Great Gatsby"]
```
