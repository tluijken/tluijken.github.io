---
title: Functional programming for modern languages
author: Thomas Luijken
date: 2023-04-05 21:35:00 +0200
categories: [Functional programming, C#]
tags: [Functional programming, C#, Lambda, Mutability, monads, currying, railway oriented programming]
---

# Functional programming for modern languages

Having spent over 15 years writing C# code focused on Object Oriented
Programming, I recently ventured into the Rust programming language, known for
its functional programming capabilities. As I started to appreciate Rust's
features, like Options, Results, and Map functions, I wondered how to integrate
functional programming principles into my C# development. This blog post will
cover how you can leverage functional programming in C# using a few simple
classes and extension methods, with insights applicable to other modern
languages that support lambda expressions or closures.


## Characteristics of Functional programming

Before we dive into code, I should first clarify a bit on the characteristics of
functional programming.

Functional programming is a programming paradigm that emphasizes immutability,
the use of pure functions, and the avoidance of side effects. Its core
characteristics include treating functions as first-class citizens, which allows
them to be passed as arguments, returned from other functions, and stored in
data structures. Functional programming also promotes the use of higher-order
functions that take other functions as input or return them as output. Recursion
is often favored over looping constructs, and the use of immutable data
structures ensures that data remains unmodified throughout the program's
execution. This paradigm ultimately leads to more predictable, concise, and
maintainable code, making it increasingly popular among developers for its
ability to manage complexity and promote robust software design.

- **Immutability**: Data remains unmodified throughout the program's execution,
  reducing the risk of unintended side effects.
- **Pure functions**: Functions depend solely on their input and produce
  consistent output without causing side effects.
- **First-class functions**: Functions can be passed as arguments, returned from
  other functions, and stored in data structures.
- **Higher-order functions**: Functions that accept other functions as input or
  return them as output, enabling powerful composition and abstraction
  techniques.
- **Recursion**: Repeated function calls to solve problems, often used as an
  alternative to looping constructs.
- **Declarative programming**: Code focuses on expressing what should be done,
  rather than detailing step-by-step procedures.
- **Lazy evaluation**: Computation is deferred until absolutely necessary,
  improving performance and avoiding unnecessary work.
- **Pattern matching**: A flexible way to destructure data and match specific
  patterns, allowing for concise and readable code.
- **Type inference**: Automatic determination of data types by the compiler,
  reducing the need for explicit type annotations.
- **Referential transparency**: Functions with the same input always produce the
  same output, making code easier to reason about and test.

## Problems with imperative programming

Let's start with a small example, converting from degrees Celcius to degrees
Fahrenheit.

```csharp
public void ConvertToFahrenheit(int degrees)
{
    var a = degrees * 9;
    var b = a / 5;
    var c = b + 32;
    Console.Writeline($"Convertered {degrees} degrees celsius to {c} degrees fahrenheit")
}
```

Notice a, b and c live longer than they need to and can be used outside of the
scope they were intended for. I could potentially use a, b or c in another part
of the code and get unexpected results.

```csharp
public void ConvertToFahrenheit(int degrees)
{
    var a = degrees * 9;
    var b = a / 5;
    var c = a + 32; // WHOOPS....
    Console.Writeline($"Convertered {degrees} degrees celsius to {c} degrees fahrenheit")
}
```

You might think: "well isn´t this obvious??", but for larger implementations I've seen
this gone wrong more then once. It would be great if we could get rid of all
intermediate values as soon as possible.

### The Map Function
The `Map` function is a function that takes a value, and a function to convert
that value to another value. It would look something like this:

```csharp
public static TTarget Map<TSource, TTarget>(this TSource source, Func<TSource, TTarget> factory) => factory(source);
```

I can now use this method to perform the same steps as above, but placing the
intermediate values out of scope as soon as possible. Because the converted
value is returned, we can just chain a new Map function to it and keep going.

```csharp
public void ConvertToFahrenheit(int degrees)
{
    var c = degrees
        .Map(d => d * 9)
        .Map(a => a / 5)
        .Map(b => b + 32);

    Console.Writeline($"Convertered {degrees} degrees celsius to {c} degrees fahrenheit")
}
```

### Making it declaritive
While the function above makes our intermediate values short lived, and
immmutable, it still it a bit difficult to read. To make it more declaritive on
what each step does, we could leverage [currying](https://en.wikipedia.org/wiki/Currying).

Currying is a technique in functional programming where a function that takes
multiple arguments is transformed into a sequence of functions, each taking a
single argument and returning another function until all arguments are supplied.

This is an example of a curried function:
```csharp
private static readonly Func<double, Func<double, double>> Multiply = x => y => y * x;
```
Which does the exact same thing as this fully written example.

```csharp
private static Func<double, double> Multiply(double x)
{
    return (double y) =>
    {
        return y * x;
    };
}
```

The function Multiply, takes in a paramater (the multiplier), and returns a new
function, what multiplies the given value with the multiplier set before.

To call a curried function, you would call it like to:
```csharp
var multiply = Multiply(2)(2);
Assert.Equal(4, multiply);
```
This may look weird at first, but what we're doing here is called partial
application. You can perform a part of the application, and re-use that part as
much as you like. Not, let this sink in. Re-using partial application could be
very powerful, depending on the use case you're working on.

```csharp
var multiplyByTwo = Multiply(2);
Assert.Equal(4, multiplyByTwo(2));
Assert.Equal(6, multiplyByTwo(3));
Assert.Equal(8, multiplyByTwo(4));
```

So, using our Map function, we could use these curried functions to apply the
transformations:

```csharp
var f = degrees                                                                                  
    .Map(x => Multiply(9)(x))                                                                                  
    .Map(x => Divide(5)(x))                                                                                    
    .Map(x => Add(32)(x));                                                                                     
```

And finally, we can use the short-hand notation C# provides to apply the curried
functions:

```csharp
var f = degrees
    .Map(Multiply(9))
    .Map(Divide(5))
    .Map(Add(32));
```

We now have a conversion from celsius to fahrenheit, using a declaritive way,
using immutable values. It is easy to read, and far less error prone. 

## A more complex example:
Let's take a bigger example, validating a dutch phone number. We're not going to
use regular expressions here, but the the sake of demonstration, we'll use an
imperative implementation.

```csharp
private static bool ValidateDutchPhoneNumberImperative(string dutchPhoneNumber)
{
    // validate length. Prefixes could be either 0, 0031 or +31
    var isValid = dutchPhoneNumber.Length is 10 or 12 or 13;
    if (!isValid)
    {
        return false;
    }
    // validate prefix 0, 0031 or +31
    isValid = isValid && dutchPhoneNumber.StartsWith("0") || dutchPhoneNumber.StartsWith("0031") || dutchPhoneNumber.StartsWith("+31");
    if (!isValid)
    {
        return false;
    }

    // validate all characters are digits (excluding the + sign)
    isValid = dutchPhoneNumber[1..].All(char.IsDigit);
    if (!isValid)
    {
        return false;
    }

    // validate numbers without prefix are 9 long
    var numberWithoutPrefix = dutchPhoneNumber.StartsWith("0031") 
        ? dutchPhoneNumber[4..] 
        : dutchPhoneNumber.StartsWith("+31") 
        ? dutchPhoneNumber[3..] 
        : dutchPhoneNumber[1..];

    isValid = numberWithoutPrefix.Length == 9;
    if (!isValid)
    {
        return false;
    }

    // The number without the prefix must not start with 0
    isValid = !numberWithoutPrefix.StartsWith("0");
    if (!isValid)
    {
        return false;
    }

    return true;
}
```

We have several steps to validate if a phone number is correct. If one of the
steps fails, we return as early as possible, to save CPU cycles.

....TO BE CONTINUED
