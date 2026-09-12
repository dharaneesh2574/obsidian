---
tags:
  - interview-prep
  - oop
  - programming
source: https://www.youtube.com/watch?v=5-9BhmP9Ufk
channel: upGrad
extracted: 2026-09-12
---

# Top 50 OOP Interview Questions and Answers

**Source:** [Top 50 OOPs Interview Questions & Answers | Object Oriented Programming Interview Questions](https://www.youtube.com/watch?v=5-9BhmP9Ufk) by upGrad  
**Length:** 38:02  
**Transcript:** English, auto-generated captions  
**Note type:** Cleaned transcript-derived study notes

## Quick revision

- The four core OOP ideas are **encapsulation, abstraction, inheritance, and polymorphism**.
- A **class** is a blueprint; an **object** is a runtime instance with state and behavior.
- **Overloading** is usually compile-time polymorphism; **overriding** enables runtime polymorphism.
- An **interface** defines a contract; an **abstract class** can combine a contract with shared state and implementation.
- Prefer OOP when its modularity and domain modeling help. Avoid forcing it onto tiny, data-oriented, low-level, or performance-critical problems.

## Cleaned transcript notes: 50 questions

### 1. What is object-oriented programming? — 00:39

Object-oriented programming (OOP) is a programming paradigm organized around objects rather than only functions. An object binds data—fields or attributes—with methods that operate on that data. OOP helps divide a large problem into smaller classes and objects and supports inheritance, polymorphism, encapsulation, and abstraction.

### 2. What are the main features of OOP? — 01:37

1. **Inheritance:** A new class can reuse and extend the attributes and behavior of an existing class.
2. **Encapsulation:** Data and the methods that operate on it are bundled together, with internal state hidden behind controlled access.
3. **Polymorphism:** Different object types can be used through a common interface while providing type-specific behavior.
4. **Abstraction:** Only essential behavior is exposed; implementation details are hidden.

Interview tip from the video: name all four and explain each in at least one sentence.

### 3. What are the advantages of OOP? — 04:20

- Modularity through encapsulation
- Code reuse through inheritance
- Easier maintenance, upgrades, troubleshooting, and debugging
- Better scalability and management of large systems
- Simplification through abstraction
- Flexible designs through polymorphism
- Higher productivity and easier team collaboration
- Natural modeling of real-world entities
- Controlled access to state
- Reuse of established design patterns such as Singleton, Factory, and Observer

### 4. What is structured programming? — 09:17

The video describes structured programming as a traditional, function-based approach. Program logic is divided into functions and developed top-down. It is suitable for problems of low to moderate complexity.

### 5. What is a class? — 09:45

A class is a logical blueprint or template for creating objects. It defines data structure, initial attribute values, and methods that implement intended behavior. The video uses `Vehicle` as an example of a class.

> [!note] Editor note
> Saying that a class “does not consume memory at runtime” is too broad. Object instances consume per-instance memory, but runtimes may also store metadata and static members for the class itself.

### 6. What is an object? — 10:12

An object is a runtime instance of a class. It has attributes or properties and methods representing behavior. Once initialized, it occupies memory and can use the variables and methods defined by its class.

### 7. Must you always create an object from a class? — 10:55

No. Static members belong to the class rather than an instance and can generally be called through the class name without creating an object.

### 8. What is a constructor? — 11:24

A constructor is a special routine used to initialize an object. It is invoked automatically when the object is created and helps ensure that the object is in a valid initial state before use.

### 9. What are the types of constructors? — 12:05

The video lists:

- Default constructor
- Copy constructor
- Static constructor
- Private constructor
- Parameterized constructor

Availability and exact meaning vary by programming language.

### 10. What is a destructor? — 12:24

A destructor releases resources associated with an object when the object is destroyed or no longer used. It is invoked as part of object destruction, has no return type, and is not overloaded in languages such as C++.

> [!note] Editor note
> Destruction and memory reclamation are language-dependent. Garbage-collected languages usually use explicit resource-management patterns rather than deterministic destructors for external resources.

### 11. What is a copy constructor? — 12:53

A copy constructor creates a new object by copying an existing object of the same class, usually during initialization. The exact moments at which it runs—for example, passing by value—depend on the language and compiler rules.

### 12. What is the difference between a class and a structure? — 13:19

The transcript introduces this question and says the differences depend on the programming language, mentioning C++ and C#, but it does not provide the actual comparison before moving on.

> [!note] Editor note
> In C++, `class` and `struct` are nearly identical except for default member access and default inheritance access. In C#, a class is a reference type and a struct is a value type, with additional restrictions on structs.

### 13. Explain inheritance with an example. — 13:43

Inheritance lets a child class acquire properties and methods from a parent class, improving reuse. For example, a general `Vehicle` base class can define shared properties, while `Car`, `Bus`, and `Truck` inherit those properties and add their own specialized behavior.

### 14. What are the limitations of inheritance? — 14:27

- Parent and child classes can become tightly coupled.
- A change in the parent may require changes in children.
- Deep hierarchies are harder to navigate and understand.
- Poorly designed inheritance can create ambiguity or unintended behavior.

### 15. What are the types of inheritance? — 14:54

- **Single:** One child inherits from one parent.
- **Multiple:** One child inherits from multiple parents; this can create ambiguity such as the diamond problem.
- **Multilevel:** A chain such as `A → B → C`.
- **Hierarchical:** Multiple children inherit from one parent.
- **Hybrid:** A combination of two or more inheritance forms.

The video notes that Java does not support multiple inheritance of classes, though similar composition can be achieved with interfaces.

### 16. What is hierarchical inheritance? — 16:48

Hierarchical inheritance occurs when multiple subclasses inherit from the same base class. Classes higher in the hierarchy usually express more general information, while descendants add more specific details.

### 17. Multiple inheritance vs. multilevel inheritance — 17:07

- **Multiple inheritance:** A class has more than one direct parent. It may introduce ambiguity and the diamond problem.
- **Multilevel inheritance:** Classes form a sequential chain. It is usually easier to understand and is widely supported in OOP languages.

### 18. What is hybrid inheritance? — 17:59

Hybrid inheritance combines two or more forms of inheritance—single, multiple, multilevel, or hierarchical—within one design. It can model complex relationships but must be managed carefully to avoid ambiguity.

### 19. What is a subclass? — 18:25

A subclass, derived class, or child class inherits state and behavior from a superclass. It can extend or modify inherited behavior and introduce its own attributes and methods.

### 20. What is a superclass? — 18:47

A superclass, base class, or parent class is the class from which other classes inherit. For example, `Vehicle` can be the superclass of `Car`, `Bus`, and `Truck`.

### 21. What is an interface? — 19:01

An interface defines a contract—a set of methods that implementing classes must provide. Code can work with objects through this common contract without depending on their position in a class hierarchy. Interfaces help achieve abstraction, loose coupling, and a form of multiple inheritance of type.

### 22. What is polymorphism? — 19:55

Polymorphism means “many forms.” A common interface can have multiple implementations, allowing different object types to respond differently to the same operation.

### 23. What is static polymorphism? — 20:10

Static polymorphism, or static binding, resolves which operation to call at compile time. The video gives method overloading and operator overloading as examples.

### 24. What is dynamic polymorphism? — 20:25

Dynamic polymorphism, or dynamic binding, resolves an overridden method call at runtime according to the object’s actual type.

### 25. What is method overloading? — 20:35

Method overloading defines multiple methods with the same name but different parameter lists. The compiler selects the appropriate method based on the arguments.

### 26. What is method overriding? — 20:56

Method overriding lets a child class replace an inherited method’s implementation while keeping a compatible signature. It is a foundation of runtime polymorphism.

### 27. What is operator overloading? — 21:11

Operator overloading gives an existing operator a type-specific meaning for user-defined types. The operator’s behavior changes according to its operands.

### 28. Overloading vs. overriding — 21:41

| Overloading | Overriding |
|---|---|
| Same method name, different parameter lists | Child class redefines inherited behavior with a compatible signature |
| Usually resolved at compile time | Usually resolved at runtime |
| Often occurs within one class | Requires an inheritance or subtype relationship |

### 29. What is encapsulation? — 22:10

Encapsulation binds data and the logic that operates on it into a single unit, such as a class. It also hides internal data and restricts access through a controlled public interface.

### 30. What is data abstraction? — 22:29

Data abstraction exposes important behavior while hiding implementation details. Users interact with what an object does without needing to understand how it does it.

### 31. How is data abstraction achieved? — 22:42

The video identifies abstract classes and interfaces as primary tools. Encapsulation also contributes by hiding internal details and exposing essential operations through public methods.

### 32. What is an abstract class? — 23:16

An abstract class is intended to be extended rather than instantiated directly. It may declare abstract methods that subclasses must implement.

> [!note] Editor note
> An abstract class can usually contain both abstract and concrete methods, plus fields and constructors—not only abstract methods.

### 33. What are access specifiers? — 23:38

Access specifiers, also called access modifiers, control the visibility of classes, methods, and data members. Common examples are `public`, `private`, and `protected`. They are an important mechanism for encapsulation.

### 34. How do you instantiate an abstract class? — 24:02

You cannot instantiate an abstract class directly. Create a concrete subclass that implements all required abstract methods, then instantiate that subclass.

### 35. What is a virtual function? — 24:20

In languages such as C++, a virtual function is declared in a base class and may be overridden in a derived class. When called through a base-class pointer or reference, the implementation belonging to the object’s runtime type is selected.

### 36. What is a pure virtual function? — 24:50

A pure virtual function declares required behavior without providing a base implementation. Derived concrete classes must implement it. In C++, a class with a pure virtual function is abstract.

### 37. Data abstraction vs. encapsulation — 25:10

| Data abstraction | Encapsulation |
|---|---|
| Focuses on **what** an object exposes | Focuses on bundling state and behavior and controlling **how** state is accessed |
| Reduces conceptual complexity | Protects internal representation |
| Commonly expressed through interfaces and abstract types | Commonly expressed through classes and access modifiers |
| Primarily a design-level concern | Primarily an implementation-level concern |

### 38. Interface vs. abstract class — 25:47

The video’s central distinction is that an interface provides a contract, while an abstract class can hold shared state, constructors, and both incomplete and complete behavior. Interfaces are useful when unrelated classes need a common capability; abstract classes are useful when related classes share implementation.

> [!note] Editor note
> Several details in the video are language- and version-specific. Modern Java interfaces, for example, may have default, static, and private methods. Always answer this question in the context of the language named by the interviewer.

### 39. What is a final variable? — 26:38

A final variable can be assigned only once. After initialization, it cannot be reassigned and is commonly used to represent a constant or immutable reference binding.

> [!note] Editor note
> In Java, a final reference cannot point to a different object, but the referenced object may still be mutable.

### 40. What is an exception? — 27:09

An exception is an event during program execution that disrupts normal control flow, often because of invalid input or an unexpected condition. A program can use exception-handling logic to recover, report the problem, or terminate gracefully.

### 41. What is exception handling? — 27:59

Exception handling is the mechanism for detecting and responding to runtime failures. Constructs such as `try` and `catch` separate error-prone operations from recovery logic so the program can continue safely or fail gracefully.

### 42. Is an error the same as an exception? — 28:26

No. The video characterizes an error as a serious, often system-level and unrecoverable problem, while an exception is a condition that application code can often catch and handle. Exact definitions vary by language.

### 43. What is a try-catch block? — 29:00

Potentially failing statements are placed in a `try` block. If an exception occurs, a matching `catch` block receives it and runs the appropriate handling logic.

### 44. What is a finally block? — 29:19

A `finally` block follows `try`/`catch` and is intended to run regardless of whether an exception occurs. It is commonly used for cleanup, such as closing files or releasing resources.

> [!note] Editor note
> “Always” has rare exceptions: abrupt process termination, power loss, or runtime failure can prevent a `finally` block from running.

### 45. What is `finalize()` used for? — 30:01

The video describes Java’s `finalize()` as a garbage-collector-invoked cleanup hook before object removal, while also warning that it is unpredictable and rarely appropriate. Prefer deterministic cleanup such as `try`-with-resources or explicit resource management.

> [!warning] Editor note
> Java finalization is deprecated for removal. Do not present `finalize()` as a recommended modern cleanup technique.

### 46. What is garbage collection, and how does it work? — 30:36

Garbage collection is automatic memory management. The runtime identifies objects that are no longer reachable or needed and reclaims their memory.

### 47. Should you always use OOP? What are its limitations? — 30:49

No. OOP has a steeper learning and design curve than simple procedural programming. Modeling everything as objects may add unnecessary complexity and overhead, especially for small tasks. Choose the paradigm that best fits the problem.

### 48. What are important OOP languages? — 31:32

The video lists:

- **Java:** Platform-independent ecosystem; web, mobile, and enterprise applications
- **C++:** High-performance, systems, games, and resource-intensive software
- **C#:** Microsoft ecosystem and Windows/.NET development
- **Python:** Multi-paradigm, readable, and widely applicable
- **Ruby:** Expressive syntax and Ruby on Rails web development
- **Swift:** Apple-platform development with a focus on safety and performance
- **Kotlin:** Concise modern language widely used for Android
- **JavaScript:** Prototype-based OOP across frontend and backend development
- **PHP:** Server-side web development with mature OOP features

### 49. What are the limitations of OOP? — 33:17

- Complex class structures can be difficult to understand and debug.
- Abstraction, indirection, dynamic dispatch, and allocation can add overhead.
- Core concepts and design patterns take time to learn.
- Full OOP architecture may be excessive for small projects.
- Abstraction may reduce direct control needed in low-level or performance-sensitive work.
- Deep inheritance can complicate versioning and backward compatibility.
- Some OOP code becomes verbose.
- Shared mutable state can make concurrent programming harder.

### 50. How does C++ support polymorphism? — 35:21

C++ supports runtime polymorphism through inheritance, virtual functions, overriding, and dynamic binding. A derived class overrides a virtual function declared by a base class. When the function is called through a base pointer or reference, C++ selects the implementation using the object’s runtime type.

C++ also supports compile-time polymorphism through function overloading, operator overloading, and templates, although the video focuses on runtime polymorphism.

## Interview answer framework

For most concept questions, use this four-part structure:

1. **Definition:** Explain the concept in one sentence.
2. **Mechanism:** Say how it works.
3. **Example:** Give a tiny class/object example.
4. **Trade-off:** Mention one limitation or when not to use it.

Example: “Encapsulation bundles state and behavior in a class and restricts direct access to internal state. A `BankAccount` can keep `balance` private and expose `deposit()` and `withdraw()`. This protects invariants, though too many getters and setters can create only superficial encapsulation.”

## High-value comparisons to rehearse

- Class vs. object
- Class vs. structure, with a language specified
- Overloading vs. overriding
- Compile-time vs. runtime polymorphism
- Abstract class vs. interface
- Abstraction vs. encapsulation
- Multiple vs. multilevel inheritance
- Error vs. exception
- Deterministic cleanup vs. garbage collection

## Extraction notes

- The source uses auto-generated English captions. Obvious caption errors were normalized, including “objectoriented” → “object-oriented,” “subass” → “subclass,” “chash” → “C#,” “cotlin” → “Kotlin,” and “tri catch” → “try-catch.”
- Promotional sections at the beginning and end were omitted because they do not contribute to the study material.
- Question 12 is announced, but its answer is absent from the available transcript; an editor note supplies a short language-specific comparison.
- Several claims are dependent on programming language and version. These are marked with editor notes rather than silently presented as universal rules.
