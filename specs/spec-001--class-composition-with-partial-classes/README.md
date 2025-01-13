# \[spec-001\] Class Composition with Partial Classes

This document describes a convention for structuring classes by splitting them into multiple parts (partial classes). It focuses on simplifying class definitions through modular organization, enhancing both maintainability and readability.

While some languages (like C#) have native partial-class features, this specification uses "partial classes" to mean splitting a class into multiple parts at the source code level—not compile time—for better organization and modularity.

## Background

Partial classes let you split a single class definition across multiple files. Each file contains a distinct portion of the class, and these parts combine to form a complete class. This approach helps manage complex classes by breaking them into smaller, more manageable pieces.

**In pseudo-code:**

```python
# partial 1
class Partial1 {
    void method1() {}
    void method2() {}
}

# partial 2
class Partial2 {
    void method3() {}
    void method4() {}
}

# composed class
class MyClass extends Partial1, Partial2 {
}

# usage
obj = new MyClass();

obj.method1();  # from Partial1
obj.method4();  # from Partial2
```

> Note: The code examples on this document are language-agnostic pseudo-code, borrowing features from Python, C#, and other languages, solely for illustration.
>

## Specification

This convention introduces a method for structuring classes using partial classes, involving three main components:

1. **Base Class**: Contains the core functionality.
2. **Partial Classes**: Add additional functionalities.
3. **Composed Class**: Combines the base class and partials into a single class.

**Example in pseudo-code:**

```python
# base class
class UserBase {
    void set_name(name: str) {}

    str get_name() {}
}

# partial 1
class WithDateOfBirth {
    void set_dob(dob: Date) {}

    Date get_dob() {}
}

# partial 2
class WithAge {
    int get_age() {}
}

# composed class
class User extends WithAge, WithDateOfBirth, UserBase {
}
```

**Usage:**

```python
# instantiate
user = new User();

# e.g., outputs 20 in Dec 2025
user.set_dob("2005-04-21");
print(user.get_age());
```

### File Structure

The recommended file structure:

```plaintext
📁 <class_name>/
├── <class_name>_base.lang       # Base class
│
├── 📁 partials/                 # Contains partial implementations
│   ├── with_<partial1>.lang     # Partial 1
│   ├── ...                      # Additional partial classes
│   └── with_<partialN>.lang     # Partial N
│
└── <class_name>.lang            # Composed class (base + partials)
```

Notes:

- `<class_name>` is the class name (e.g., `User`).
- `.lang` represents the language-specific extension (e.g., `.py`, `.cs`).
- Adjust case conventions to match your language's standards.

### Name Convention

Follow these naming conventions for classes and files:

- **Base classes** are named with a `Base` suffix (e.g., `UserBase`)
- **Partial classes** start with `With` prefix (e.g., `WithAge`, `WithDateOfBirth`)
- **Composed classes** use the plain name without prefix/suffix (e.g., `User`)

### One-to-One Interface Mapping

For stronger type safety and consistent method signatures, each class should have a corresponding interface:

```plaintext
📁 <class_name>/
├── ...
├── <class_name>_base_interface.lang     # Interface for the base class
│
├── 📁 partials/
│   ├── with_<partial1>_interface.lang   # Interface for Partial 1
│   ├── ...
│   └── with_<partialN>_interface.lang   # Interface for Partial N
│
└── <class_name>_interface.lang          # Composed interface
```

Interfaces are named by appending `Interface` to the class name (e.g., `UserBaseInterface`, `WithAgeInterface`, `UserInterface`).

## Base Class

The **base class** holds essential methods or can be empty if you prefer moving all methods to partials.

### Base Class Example

Let's define the interface for the base class:

```python
# file: user_base_interface.lang
interface UserBaseInterface
{
    void set_name(name: str);

    str get_name();
}
```

The concrete class must import its interface and rename it to `ImplementsInterface`:

```python
# file: user_base.lang
"./user_base_interface" as ImplementsInterface;

class UserBase implements ImplementsInterface
{
    str _name;

    void set_name(name: str) {
        self._name = name;
    }

    str get_name() {
        return self._name;
    }
}
```

### Empty Base Class Example

```python
# file: user_base.lang
"./user_base_interface" as ImplementsInterface;

class UserBase implements ImplementsInterface
{
    # intentionally empty
}
```

## Partials

### Partial Example

A partial **interface** must **extend** the interfaces of its dependencies. All partials depend on the base class.

```python
# file: partials/with_date_of_birth_interface.lang

# base dependency
"../user_base_interface" as UserBaseInterface;

interface WithDateOfBirthInterface extends UserBaseInterface
{
    void set_dob(dob: Date);

    Date get_dob();
}
```

A **concrete** partial class must **implement** its own interface.

```python
# file: partials/with_date_of_birth.lang

# self interface
"./with_date_of_birth_interface" as ImplementsInterface;

class WithDateOfBirth implements ImplementsInterface
{
    Date _dob;

    void set_dob(dob: Date) {
        self._dob = dob;
    }

    Date get_dob() {
        return self._dob;
    }
}
```

### Partial with Dependencies

If a partial depends on other partials, its interface should **extend** those partials' interfaces:

```python
# file: partials/with_age_interface.lang

# base dependency
"../user_base_interface" as UserBaseInterface;

# this partial depends on "with_date_of_birth"
"../with_date_of_birth_interface" as WithDateOfBirthInterface;

interface WithAgeInterface
    extends WithDateOfBirthInterface,
            UserBaseInterface
{
    int get_age();
}
```

Its concrete implementation:

```python
# file: partials/with_age.lang
"./with_age_interface" as ImplementsInterface;

class WithAge implements ImplementsInterface
{
    int get_age() {
        today = Date::today();

        return self.get_dob().diff_from(today).years();
    }
}
```

## Composed Class

The **composed class** extends the base plus all partials. It remains empty, serving as the "glue" that ties everything together.

### Composed Class Example

First, let's define the composed interface that combines all other interfaces:

```python
# file: user_interface.lang

# base dependency
"./user_base_interface" as UserBaseInterface;

# partial dependencies
"./partials/with_date_of_birth_interface" as WithDateOfBirthInterface;
"./partials/with_age_interface" as WithAgeInterface;

interface UserInterface
    extends WithAgeInterface,
            WithDateOfBirthInterface,
            UserBaseInterface
{
    # intentionally empty
}
```

Then, implement the composed class that combines all concrete implementations:

```python
# file: user.lang

# self interface
"./user_interface" as ImplementsInterface;

# concrete implementations
"./user_base" as UserBase;
"./partials/with_date_of_birth" as WithDateOfBirth;
"./partials/with_age" as WithAge;

class User
    extends WithAge,
            WithDateOfBirth,
            UserBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

## Usage Example

Now that all the components are in place, let's use it:

```python
# reference the composed class
"./user" as User

user = new User();

# set basic information
user.set_name("John Smith");
user.set_dob("1990-05-15");

# use methods from different partials
print(user.get_name());  # "John Smith"
print(user.get_dob());   # "1990-05-15"
print(user.get_age());   # "34" (as of 2025)

# verify type assignments
assert user instanceof UserInterface            # true
assert user instanceof WithDateOfBirthInterface # true
assert user instanceof WithAgeInterface         # true
assert user instanceof UserBaseInterface        # true
```

## Additional Considerations

1. **Import Order**
    - When combining multiple interfaces, reference them from higher-level (base) to lower-level (partials).
    - The same applies for concrete classes that implement multiple partials.
2. **External Sources**
    - You can also create partials that extend external or library classes.
    - Simply import the external class and extend it alongside the partial interface.
3. **Keep Composed Classes and Their Interfaces Empty**
    - They should only serve as an aggregator of functionality.
    - Place any additional logic in a separate partial.
    - Failing this rule defeats the purpose of splitting the class into partials and centralizes complexity in a single file.
