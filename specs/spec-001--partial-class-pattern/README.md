# \[spec-001\] Partial Class Pattern

This document describes the "Partial Class Pattern"—a convention for organizing classes by dividing them into multiple parts. The pattern simplifies class definitions through modular organization, making code more maintainable and readable.

While some languages (like C#) have native partial-class features, this specification uses "partial classes" to mean splitting a class into multiple parts at the **source code level**—not compile time—for better organization and modularity.

## Background

Partial classes let you split a single class definition across multiple files. Each file contains a distinct portion of the class, and these parts combine to form a complete class.

**In pseudo-code:**

```python
# partial 1
class Partial1 extends Base {
    void method1() {}
    void method2() {}
}

# partial 2
class Partial2 extends Base {
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

1. **Base Class**: An empty marker class that serves as the root parent for all partials.
2. **Partial Classes**: Contain all functionality, each extending the base class.
3. **Composed Class**: Combines the base plus all partials into a single class.

**Example in pseudo-code:**

```python
# base class (empty marker)
class UserBase {
    # intentionally empty
}

# partial 1
class WithDateOfBirth extends UserBase {
    void set_dob(dob: Date) {}
    Date get_dob() {}
}

# partial 2
class WithAge extends UserBase {
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

```
📁 <class_name>/
├── <class_name>_base.lang       # Empty base class (marker)
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

```
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

The **base class** must always be empty, serving only as a marker interface that all partials extend from.

### Base Class Example

Let's define the interface for the base class:

```python
# file: user_base_interface.lang
interface UserBaseInterface
{
    # intentionally empty
}
```

The concrete class must import its interface and rename it to `ImplementsInterface`:

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

A partial **interface** must **extend** the base interface:

```python
# file: partials/with_date_of_birth_interface.lang

# base interface
"../user_base_interface" as UserBaseInterface;

interface WithDateOfBirthInterface extends UserBaseInterface
{
    void set_dob(dob: Date);

    Date get_dob();
}
```

A **concrete** partial class must **implement** its own interface:

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

If a partial depends on other partials, its interface should **extend** those partials' interfaces while still extending the base interface:

```python
# file: partials/with_age_interface.lang

# base interface
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

# self interface
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

# base interface
"./user_base_interface" as UserBaseInterface;

# partials interfaces
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

# base
"./user_base" as UserBase;

# partials
"./partials/with_date_of_birth" as WithDateOfBirth;
"./partials/with_age" as WithAge;

# self interface
"./user_interface" as ImplementsInterface;

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
user.set_dob("1990-05-15");

# use methods from different partials
print(user.get_dob());   # "1990-05-15"
print(user.get_age());   # "34" (as of 2025)

# verify type assignments
assert user instanceof UserInterface            # true
assert user instanceof WithDateOfBirthInterface # true
assert user instanceof WithAgeInterface         # true
assert user instanceof UserBaseInterface        # true
```

## Advanced Usage

### Method Overloading in Partials

When a partial needs to modify behavior from another partial, use the language's native method overloading features.

**Example scenario:** Let's say we want to create a specialized age calculation that rounds down to the nearest year:

```python
# file: partials/with_conservative_age.lang

# self interface
"./with_conservative_age_interface" as ImplementsInterface;

class WithConservativeAge implements ImplementsInterface
{
    # Overrides get_age() from WithAge
    int get_age() {
        age = super.get_age();

        return Math.floor(age);
    }
}
```

The composed class would then use this partial last in the inheritance chain to ensure its version takes precedence:

```python
class User
    extends WithConservativeAge,  # last to override
            WithAge,
            WithDateOfBirth,
            UserBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

### Extending External Sources

You can use partials to extend code from external libraries.

Simply import the external class and extend it alongside the partial interface.

**Example:**

Let's say we want to extend a library called "DateTime" that provides a concrete class "DateTime" and an interface "IDateTime" (as most libraries do).

First, create a base that wraps the external class.

```python
# file: datetime_base_interface.lang

# library's interface, renamed to our convention
"datetime_library/idatetime" as "DateTimeInterface";

interface DateTimeBaseInterface extends DateTimeInterface
{
    # intentionally empty
}
```

Then, create the concrete class for the base:

```python
# file: datetime_base.lang

# library's concrete class
"datetime_library/datetime" as "DateTime";

# self interface
"./datetime_interface" as ImplementsInterface;

class DateTimeBase
    extends DateTime
    implements ImplementsInterface
{
    # intentionally empty
}
```

Now, use these as a normal base to implement partials and the composed class.

```
📁 datetime/
├── datetime_base.lang   # Base class
├── datetime_base_interface.lang
│
├── 📁 partials/
│   ├── with_timezone.lang
│   ├── with_iso8601_format.lang
│   │
│   └── ... # interfaces
│
├── datetime.lang        # Composed class
└── datetime_interface.lang
```

Create the interface for the composed class just like any other:

```python
# file: datetime_interface.lang

# base interface
"./datetime_base_interface" as DateTimeBaseInterface;

# partials interfaces
"./partials/with_timezone_interface" as WithTimezoneInterface;
"./partials/with_iso8601_interface" as WithIso8601Interface;

interface DateTimeInterface
    extends WithIso8601Interface,
            WithTimezoneInterface,
            DateTimeBaseInterface
{
    # intentionally empty
}
```

The concrete composed class follows the same pattern:

```python
# file: datetime.lang

# self interface
"./datetime_interface" as ImplementsInterface;

# base
"./datetime_base" as DateTimeBase;

# partials
"./partials/with_timezone" as WithTimezone;
"./partials/with_iso8601" as WithIso8601;

class DateTime
    extends WithIso8601,
            WithTimezone,
            DateTimeBase
    implements ImplementsInterface
{
    # intentionally empty
}
```

Finally, use the composed class instead of the external library:

```python
# import our composed class
"./datetime" as DateTime;

dt = new DateTime();

# use methods from library
dt.add_days(5);
dt.set_hour(14);

# use methods from partials
dt.to_timezone("UTC");
dt.to_iso8601();  # "2025-01-20T14:00:00Z"
```

## Additional Considerations

1. **Import Order**
    - When combining multiple interfaces, reference them from highest level (base) to lowest level (partials).
    - Follow the same order when implementing multiple partials in concrete classes.
2. **Keep Composed Classes and Their Interfaces Empty**
    - Use them only to aggregate functionality.
    - Move any additional logic into a separate partial.
    - Adding logic to composed classes undermines the benefits of partials and creates a single point of complexity.
3. **Concrete Classes**
    - Only the composed concrete class should use concrete classes from partials and base.
    - Neither partials nor other code should directly reference concrete classes.
