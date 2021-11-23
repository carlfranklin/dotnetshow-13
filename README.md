# Classes, Records, and Structs. Oh My!

Functional languages like Haskell, SML, Clojure, Scala, Erlang, Clean, F#, and the LISP family have always used immutable objects by default. Immutable means you can set the value once, but that's it. If you need to change it, you have to create a new object. If the object is of a complex type with properties, the new object can be created from the existing one but with the changes made. This language feature pushed the C# team into creating a new `record` type. How do records differ from classes or structs? Why do we need three fundamental types from which we can create objects? When would we use one over the other? I hope to answer these questions. But first, we need to look at where we were before C# 9, and focus a bit on some fundamentals.

## Immutability

Immutability can be useful when you need a data-centric type to be thread-safe or you're depending on a hash code remaining the same in a hash table.  Immutability isn't appropriate for all data scenarios, however. Entity Framework Core, for example, doesn't support updating with immutable entity types.

Before C#9, if we wanted to create an object that couldn't be changed, we had to make all the properties read-only (without setters) and require passing in the property values in the constructor.

```c#
using static System.Console;

var person = new Person("Carl", "Franklin");

//person.FirstName = "Ben";

WriteLine($"{person.FirstName} {person.LastName}");

public class Person
{
    public Person(string FirstName, string LastName)
    {
        this.FirstName = FirstName;
        this.LastName = LastName;
    }

    public string FirstName { get; } = "";
    public string LastName { get; } = "";
}
```

The C# team, recognizing the benefit of immutability, introduced new language features in C# 9 (2020), and improved on them in C# 10 (2021). 

### Initializers

Starting with C# 9 we can now use the `init` keyword in place of the `set` keyword to express that we want a property to be immutable. 

```c#
using static System.Console;

var person = new Person() { FirstName = "Carl", LastName = "Franklin" };

// This will break the build
person.LastName = "Jung";

WriteLine($"{person.FirstName} {person.LastName}");

public class Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
}
```

### Structs

Just for fun, let's change `Person` from a `class` to a `struct`

```c#
public struct Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
}
```

As you can see, structs can have initializers too.

#### Classes vs Structs

A `class` creates a **reference type**, and a `struct` creates a **value type**. To really understand the difference we have to go back to some fundamentals. Stacks and Heaps. You see, value types are created on the **stack** whereas reference types are created on the **heap**.

##### What is a Stack?

<img src="https://user-images.githubusercontent.com/1486348/143114517-d3444400-c217-4a79-86c1-73050948763a.png" style="zoom:50%;" />

As in real life, a stack is a pile of things, one on top of the other. In computer lingo, it's a stack of values and pointers. Every thread has it's own stack. Value types are created on the stack of the executing thread. Only the top value can be accessed, exposing the next value. When you add a value to the stack, you are **push**ing it. When you retrieve a value from the stack, you are **pop**ping it. `struct` is a value type, as are `bool`, `byte`, `char`, `decimal`, `double`, `enum`, `float`, `int`, `long`, `sbyte`, `short`, `uint`, `ulong`, and `ushort`

##### What is a heap?

<img src="https://user-images.githubusercontent.com/1486348/143114612-02074345-4ff5-48f8-a9e3-74e64aa83187.png" style="zoom:50%;" />

As in real life, a heap is a pile of things, accessible randomly. In computer lingo, it's an area of memory belonging to an application. It is accessible by all threads. Think of it as global memory. Reference types are created on the heap, as are value types that are defined inside a reference type. Reference types are `interface`, `delegate`, `object`, and `string`. `string` is an immutable reference type. Any time you change the value, a new string object is created. That's why `System.Text.StringBuilder` exists.

```
// Here we go. Call SomeMethod();
SomeMethod();

void SomeMethod()
{
    // The stack is empty before statement 1
    int x = 101;
    
    // After statement 1, x has been pushed onto the stack.

	int y = 102;
        
    // After statement 2, y has been pushed onto the stack.

	SomeClass cls1 = new SomeClass();
	
	// After statement 3, cls1 has been added to the heap,
	//    and a reference to cls1 has been pushed onto the stack.
}

public class SomeClass
{
    // Since this value type (x) is declared
    // inside a reference type (SomeClass),
    // it will be created on the heap, not the stack.
    public int x { get; set; }     
}
```

##### Code pointers on the stack

Before calling a method, the current code pointer is pushed onto the stack by the .NET runtime, so that it can resume where it left off before the method was called.

```c#
using static System.Console;

// 1) push the arguments onto the stack (Year, Month, Day)
//    All are value types, so the values are pushed onto the stack.
int Age = CalculateAge(1985, 3, 14);

// 8) Pop Age off the stack to access it.
WriteLine($"My age is {Age}");

int CalculateAge(int Year, int Month, int Day)
{
    // 2) Pop the parameter values off the stack to access them

    // 3) DateTime is a value type, so Birthday is
    //    pushed onto the stack
    DateTime Birthday = new DateTime(Year, Month, Day);

    // 4) TimeSpan is a value type pushed onto the stack
    TimeSpan TimeRange = DateTime.Now.Subtract(Birthday);

    // 5) Push days on the stack
    int Days = Convert.ToInt32(TimeRange.TotalDays);

    // 6) Push age on the stack
    int Age = Days / 365;

    // 7) Clear all the values we've added to the stack
    //    and push age on the stack as the return value.
    return Age;
}
```

This is a good tutorial about the stack and the heap, for further reading:

https://dotnettutorials.net/lesson/stack-and-heap-dotnet/

#### When to use a struct vs a class

You probably are wondering when you should use a `struct` vs a `class`. Simply put, use a class when you need a reference to a single source of truth, such as a complex hierarchy of data. Use a struct when you do NOT need references. Common structs in the .NET Framework are `Point` (x, y), `Rectangle` (x1, y1, x2, y2) and `Color`. These are small bits of information that contain value types to express a set of values that make up a thing. Typically they do not need to exist outside of your code scope. For all other situations, a class makes more sense.

#### Recap

So far we know that instantiating a `class` yields a reference type. The object itself is stored on the heap, but the reference is stored on the stack of the executing thread. We also know that a `struct`, being a value type is created on the stack. `struct` should be used to express sets of value types that together represent something like a `Point` or a `Shape`. For all other situations, a class is the right choice. But, classes have some issues, as we will discover now.

### Equality

Let's try comparing two objects (from a class) that have the same internal values:

```c#
using static System.Console;

var person1 = new Person() { FirstName = "Carl", LastName = "Franklin" };
var person2 = new Person() { FirstName = "Carl", LastName = "Franklin" };

// this will show "person1 = person2? False"
WriteLine($"person1 = person2? {Equals(person1, person2)}");

public class Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
}
```

In this case, two different objects are created on the heap. Equality means that both the variables `person1` and `person2` point to (refer to) the same object on the heap, which they do not. If you simply created a second reference to a single class, they would be equal:

```c#
using static System.Console;

var person1 = new Person() { FirstName = "Carl", LastName = "Franklin" };
var person2 = person1;

// this will show "person1 = person2? True"
WriteLine($"person1 = person2? {Equals(person1, person2)}");

public class Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
}
```

One of the benefits of `struct` is that when you compare two of them, they will be equal if the values inside the struct are equal:

```c#
using static System.Console;

var person1 = new Person() { FirstName = "Carl", LastName = "Franklin" };
var person2 = new Person() { FirstName = "Carl", LastName = "Franklin" };

// this will show "person1 = person2? True"
WriteLine($"person1 = person2? {Equals(person1, person2)}");

public struct Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
}
```

You can override the Equals operator of a class to compare the values, but you have to test each value, which can be daunting, especially if there is a complex hierarchy:

```c#
using static System.Console;

var person1 = new Person() { FirstName = "Carl", LastName = "Franklin" };
var person2 = new Person() { FirstName = "Carl", LastName = "Franklin" };

// this will show "person1 = person2? True"
WriteLine($"person1 = person2? {Equals(person1, person2)}");

public class Person
{
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";

    public override bool Equals(object? obj)
    {
        var other = obj as Person;  
        if (other == null) return false;
        return (Equals(other.FirstName, this.FirstName)
            && Equals(other.LastName, this.LastName));
    }

    public override int GetHashCode()
    {
        return base.GetHashCode();
    }
}
```

### Introducing Records

In C# 9, Microsoft introduced a new base type called `record`. A `record` can be a `class` (Reference Type) or a `struct`, but is a `class` by default.  For all record types, including `record struct` and `readonly record struct`, two objects are equal if they are of the same type and store the same values.

Another feature of `record` is that you can define properties with positional syntax, meaning you don't need to use `{get; set;}` or `{get; init;}` If you define them with positional syntax, as in this example, it's the same as using `{get; init;}` but without the additional ceremony. Notice how you can also create a new record using positional syntax. Because `FirstName` is defined first, the compiler knows that `"Carl"` is mapped to the `FirstName` property.

```c#
using static System.Console;

var person1 = new Person("Carl", "Franklin");
var person2 = new Person("Carl", "Franklin");

// this will show "person1 = person2? True"
WriteLine($"person1 = person2? {Equals(person1, person2)}");

public record Person(string FirstName, string LastName);
```

As I mentioned, a `record` is a reference type (class) by default. In C# 10, however, you can specify `class` or `struct`:

```c#
// Person defines a reference type
public record class Person(string FirstName, string LastName);
```

```c#
// Person defines a value type
public record struct Person(string FirstName, string LastName);
```

Since we are defining the properties of `Person` with positional syntax, it is immutable by default:

```c#
var person = new Person("Carl", "Franklin");

// This will break the build.
person.FirstName = "Ben";

public record class Person(string FirstName, string LastName);
// We could also say 'public record Person(string FirstName, string LastName);'
//    but we added the class keyword for clarity.
```

However, if we define the `record` as a `struct`, it will NOT be immutable

```c#
var person = new Person("Carl", "Franklin");

// This will NOT break the build
person.FirstName = "Ben";

public record struct Person(string FirstName, string LastName);
```

To make a record struct immutable, you can use the init keyword. Note that we can no longer create the struct with positional syntax:

```c#
var person = new Person() { FirstName = "Carl", LastName = "Franklin" };

// This will break the build
person.FirstName = "Ben";

public record struct Person
{
    public string FirstName { get; init;}
    public string LastName { get; init;}
}
```

#### 'with' expression

C# 9 also introduced a new expression, `with` that can be used to create a new record from an existing record, but changing one or more properties in the process. We call this **nondestructive mutation**.

```c#
using static System.Console;

var person1 = new Person("Carl", "Franklin");
var person2 = person1 with { FirstName = "Ben"};

// This will output "Ben Franklin"
WriteLine($"{person2.FirstName} {person2.LastName}");

public record Person(string FirstName, string LastName);
```

#### Built-in formatting for display

One other nice feature of `record` is the `ToString()` method actually returns a nicely-formatted value representation:

```c#
using static System.Console;

var person1 = new Person("Carl", "Franklin");
var person2 = person1 with { FirstName = "Ben"};

// This will output "Person { FirstName = Ben, LastName = Franklin }"
WriteLine(person2);

public record Person(string FirstName, string LastName);
```

#### Inheritance for record class types

A record can inherit from another record. However, a record can't inherit from a class, and a class can't inherit from a record. There's no generic constraint that requires a type to be a record. Records satisfy either the `class` or `struct` constraint.

More information is available in the documentation at http://records.thedotnetshow.com

## When to use what?

### class vs record class

So you know that you want a reference type. Use a `record` if you want the equality feature of a struct without having to override `Equals` and/or you want to enforce immutability with less ceremony. Otherwise, a class is sufficient.

### struct vs record struct

A `record struct` is exactly the same as a `struct` with the following benefits:

- Can be defined with positional syntax.
- Can be nondestructively mutated using a`with` expression.
- Performance. In some benchmarks, a **record struct** is 20 times faster than a regular **struct**. 

So, there's no downside to using `record struct` over `struct`. 

