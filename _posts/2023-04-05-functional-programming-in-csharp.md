---
title: Functional programming for modern languages
author: Thomas Luijken
date: 2023-04-05 21:35:00 +0200
categories: [Functional programming, C#]
tags: [Functional programming, C#, Lambda, Mutability, monads, currying, railway oriented programming]
---

Having spent over 15 years writing C# code focused on Object Oriented
Programming, I recently ventured into the Rust programming language, known for
its functional programming capabilities. As I started to appreciate Rust's
features, like Options, Results, and Map functions, I wondered how to integrate
functional programming principles into my C# development. This blog post will
cover how you can leverage functional programming in C# using a few simple
classes and extension methods, with insights applicable to other modern
languages that support lambda expressions or closures.

All code for this blogpost can be found [in this Github
repository](https://github.com/tluijken/functional-csharp-examples)

# Characteristics of Functional programming

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

# Problems with imperative programming

Let's start with a quick example: converting Celsius to Fahrenheit.

```csharp
public void ConvertToFahrenheit(int degrees)
{
    var a = degrees * 9;
    var b = a / 5;
    var c = b + 32;
    Console.Writeline($"Convertered {degrees} degrees celsius to {c} degrees fahrenheit")
}
```

It's worth noting that a, b, and c have a longer lifespan than necessary and
could be unintentionally used elsewhere in the code. This could result in
unexpected outcomes, so it's important to be cautious when reusing variables
outside of their intended scope.

```csharp
public void ConvertToFahrenheit(int degrees)
{
    var a = degrees * 9;
    var b = a / 5;
    var c = a + 32; // WHOOPS....
    Console.Writeline($"Convertered {degrees} degrees celsius to {c} degrees fahrenheit")
}
```

While it may seem obvious, I've seen this mistake happen more than once in
larger code implementations. To prevent unexpected outcomes, it's best to
eliminate intermediate values as soon as they are no longer needed. This helps
to avoid unintentional reuse of variables outside of their intended scope.

# Scoping and mutability
Scoping and mutability are important concepts to keep in mind when working with
variables in a program. As for the Map function, it takes a value and a
conversion function, which enables the transformation of values in a clear and
concise way. To illustrate, consider the following example:

```csharp
public static TTarget Map<TSource, TTarget>(this TSource source, Func<TSource, TTarget> factory) => factory(source);
```

By utilizing this method, I can perform the same steps as before while ensuring
that intermediate values are scoped appropriately. Since the converted value is
returned, we can immediately chain a new Map function to it and continue
processing without having to worry about reusing variables unintentionally.

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

# Making it declaritive
Although the previous function ensures that intermediate values are short-lived
and immutable, it can still be challenging to read and understand. However, by
making it more declarative, we can shift our focus to what we want to accomplish
rather than how to do it. One approach to achieving this is by utilizing a
technique called [currying](https://en.wikipedia.org/wiki/Currying).

Currying transforms a multi-argument function into a series of single-argument
functions that return another function, until all necessary arguments are
provided.

To illustrate, consider the following example of a curried function:

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

The Multiply function takes a parameter (the multiplier) and returns a new
function that multiplies a given value by the specified multiplier. When calling
a curried function, you can do so by providing the arguments one at a time, like
this:

```csharp
var multiply = Multiply(2)(2);
Assert.Equal(4, multiply);
```

Although it may seem unconventional at first, what we're doing here is actually
an example of partial application. With partial application, you can supply some
of the arguments ahead of time and reuse that partial application as many times
as needed. This can be a powerful technique depending on the use case you're
working on, so take a moment to let it sink in.

```csharp
var multiplyByTwo = Multiply(2);
Assert.Equal(4, multiplyByTwo(2));
Assert.Equal(6, multiplyByTwo(3));
Assert.Equal(8, multiplyByTwo(4));
```

Now, let's consider how we could use these curried functions with our `Map`
function to apply the necessary transformations:

```csharp
var f = degrees                                                                                  
    .Map(x => Multiply(9)(x))                                                                                  
    .Map(x => Divide(5)(x))                                                                                    
    .Map(x => Add(32)(x));                                                                                     
```

To simplify the process even further, we can utilize the shorthand notation that
C# provides to apply the curried functions:

```csharp
var f = degrees
    .Map(Multiply(9))
    .Map(Divide(5))
    .Map(Add(32));
```

By implementing a declarative approach that utilizes immutable values, we now
have a clear and concise conversion from Celsius to Fahrenheit that is easy to
read and far less prone to errors.

> But that seems like a lot of work, for such a simple task Thomas.

While it may seem like a lot of work for such a simple task, it's important to
keep in mind that the functions we created can be packaged and reused across
multiple projects. This makes the investment of time and effort in creating them
well worth it in the long run, as it can lead to greater efficiency and
consistency in your codebase.

# Functional validation
Now, let's consider a larger example: validating a Dutch phone number. While we
won't be using regular expressions in this example, we'll provide an imperative
implementation to illustrate the concept.

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
To validate a phone number, there are several steps involved. If any of these
steps fail, it's important to return as early as possible to save CPU cycles.
However, the imperative approach used to accomplish this focuses primarily on
how to validate the phone number, rather than what we actually want to achieve.
This can result in a lot of noise, with if checks and return statements that
make the code difficult to read.

To address this issue, we can utilize a
[higher-order](https://en.wikipedia.org/wiki/Higher-order_function) function and
pass in a set of validation functions to aggregate over. This will allow us to
focus more on the what of the problem, rather than the how. 

First, we'll need to split up the various validations into named functions.

```csharp
private static readonly Func<string, bool> ValidateAllDigits =
    dutchPhoneNumber => dutchPhoneNumber[1..].All(char.IsDigit);

private static readonly Func<string, bool> ValidatePhoneNumberLength =
    dutchPhoneNumber => dutchPhoneNumber.Length is 10 or 12 or 13;

private static readonly Func<string, bool> ValidatePhoneNumberPrefix = dutchPhoneNumber =>
    dutchPhoneNumber.StartsWith("0") || dutchPhoneNumber.StartsWith("0031") || dutchPhoneNumber.StartsWith("+31");

private static readonly Func<string, bool> ValidateNumberWithoutPrefixLength =
    dutchPhoneNumber => SubtractPrefix(dutchPhoneNumber).Length == 9;

private static readonly Func<string, bool> ValidateNumberWithoutPrefixIsNotZero =
    dutchPhoneNumber => !SubtractPrefix(dutchPhoneNumber).StartsWith("0");

private static readonly Func<string, string> SubtractPrefix = dutchPhoneNumber =>
    dutchPhoneNumber.StartsWith("0031") 
        ? dutchPhoneNumber[4..] 
        : dutchPhoneNumber.StartsWith("+31") 
            ? dutchPhoneNumber[3..] 
            : dutchPhoneNumber[1..];

```

This accomplishes the same result as the previous imperative approach, but with
the validations split up into named functions. 

Now, we can further improve this by using a higher-order function to aggregate
over the validation functions. We can create a Validate extension method that
takes in all of the validation functions as arguments.

```csharp
public static bool Validate<T>(this T @this, params Func<T, bool>[] predicates) => predicates.All(p => p(@this));
```

The All method used on the predicates is a useful feature because it returns
false as soon as one of the predicates fails to return true, which is equivalent
to the early return false statements used in the imperative implementation. By
using the `Validate` method on the phone number itself, we can easily validate the
number using a declarative approach with more readable code that focuses on what
we want rather than how we want to validate it.

```csharp
private static bool ValidateDutchPhoneNumberFunctional(string dutchPhoneNumber) =>
    dutchPhoneNumber.Validate(
        ValidatePhoneNumberLength,
        ValidatePhoneNumberPrefix,
        ValidateAllDigits,
        ValidateNumberWithoutPrefixLength,
        ValidateNumberWithoutPrefixIsNotZero
    );
```

Adding additional validations becomes much easier as we can simply add them to
the list of validations without worrying about the control flow of an imperative
implementation.

Anonymous functions can also be used in place of named functions for more
concise code. Here's an example using anonymous functions to validate a
username.

```csharp
[Theory]
[InlineData("Thomas Luijken", true)]
[InlineData("Justin Bieber", false)]
[InlineData("Ju", false)]
[InlineData("This is a username that is way too long", false)]
public void ValidateUsername(string userName, bool valid)
{
    var validateUserName = userName.Validate(
        un => !string.IsNullOrWhiteSpace(un),
        un => un.Length >= 3,
        un => un.Length <= 20,
        un => !un.Equals("Justin Bieber"));
    Assert.Equal(valid, validateUserName);
}
```

# Monads

In many cases, we need to convert data in a sequence of steps into a different
form. Consider this `Person` class, for example

```csharp
public record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
    public Person Spouse { get; init; }
}
```

If we have a person record and we want to get the first name of the spouse, we
can access it like this:

```csharp
public string GetSpouseFirstName(Person person)
{
    var spouse = person.Spouse;
    var spouseFirstName = spouse.FirstName;
    return spouseFirstName
}
```

However, depending on the language version and the nullable settings for your
project, either of these values can be null, despite what the compiler may
suggest, and the operation would fail at runtime. To handle this, we would need
to add null checks.

```csharp
public string GetSpouseFirstName(Person person)
{
    if (person is not null) {
        var spouse = person.Spouse;
        if (spouse is not null)
        {
            var spouseFirstName = spouse.FirstName;
            if (spouseFirstName is not null)
            {
                return spouseFirstName
            }
        }
    }
    return string.Empty;
}
```
This approach adds a lot of clutter to our code and shifts the focus away from
what we're trying to accomplish. To avoid this, we can use a design pattern in
which the pipeline implementation is abstracted by wrapping a value in a type.
This pattern is called a
[Monad](https://en.wikipedia.org/wiki/Monad_(functional_programming)), which you
can learn more about in the functional programming paradigm. Within functional
programming, you'll find types such as Result or Option (also called Maybe) that
wrap the values for each result in a pipeline.

Here's what our `Option` type looks like:

```csharp
public abstract record Option<T>
{
    // by using the implicit operator, we can directly convert a value of T to an Option<T> being either Some<T> or None<T>
    public static implicit operator Option<T>(T value) => value is null ? new None<T>() : new Some<T>(value);
}
```

For our `Option` type, I will define 2 subtypes: Some and None:

```csharp
public record Some<T>(T Value) : Option<T>
{
    // by using the implicit operator, we can directly convert a value of T to an Some<T>
    public static implicit operator Some<T>(T value) => new(value);
    
    // Additionally, we can also convert an Some<T> to a T implicitly
    public static implicit operator T(Some<T> @this) => @this.Value;
}

// Hello darkness, my old friend
public record None<T> : Option<T>;
```

A value can be implicitly converted to Some if the value is not null, and can be
implicitly converted to None if the value is null.

Now, let's extend our mapping function a bit:

```csharp
public static Option<TTarget> Map<TSource, TTarget>(this Option<TSource> source, Func<TSource, TTarget> factory) =>
    source switch
    {
        Some<TSource> some => TryCreate(() => factory(some.Value)),
        _ => new None<TTarget>()
    };

private static Option<T> TryCreate<T>(Func<T> func)
{
    try
    {
        return new Some<T>(func());
    }
    catch
    {
        return new None<T>();
    }
}
```

In this case, we first check if the value we got is actually a value or if it's
None. If we got a None value, we'll simply return another None.

If we got a Some value, we can now safely attempt to map the value by using the
provided factory method. If the mapping fails, we'll return None. Otherwise,
we'll return a new Some value wrapping the mapped result.

By abstracting this logic away into the Map method, we can now chain our calls
without worrying about control flow or null checks. This allows us to focus on
the data transformations we want to achieve, rather than the boilerplate code
for handling null values.

```csharp
public string GetSpouseFirstName(Person person) => 
    person.Map(a => a.Spouse)
          .Map(spouse => spouse.FirstName);
```

If the person is `None`, or the `spouse` is `None`, or the `firstname` of the
spouse is `None`, we will just return `None`. If all values are set, the
firstname of the spouse will be returned. 

We can continue chaining these operations to process the data as needed.

```csharp
public string GetSpouseFirstName(Person person) => 
    person
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(a => a.Spouse)
        .Map(spouse => spouse.FirstName);
```

The code in the previous paragraph illustrates a situation where we get the
spouse of the spouse of the spouse, and so on, until we eventually encounter a
null value somewhere in the chain. This approach is also known as "[Railway
oriented programming](https://fsharpforfunandprofit.com/rop/)", which emphasizes
handling success and failure paths in a linear manner. 


# Performing intermediate logic

At times, you may want to include additional logic in the pipelines mentioned
earlier. For instance, you may want to log an intermediate value before
proceeding with further operations on the same result.

```csharp
public void TestTeeImperative(Person person)
{
    var spouse = person.Spouse;
    if (spouse is not null)
    {
        _testOutputHelper.WriteLine($"{spouse.FirstName} is {spouse.Age} years old");
        if (spouse.FirstName is not null)
        {
            if (spouse.FirstName.Length > 0)
            {
                Assert.Equal("Jane", spouse.FirstName);
            }
        }
        else
        {
            throw new Exception("Spouse's first name is null");
        }
    }
    else
    {
        _testOutputHelper.WriteLine("No spouse");
    }
}
```
Functional programming can be used to accomplish this goal by creating a function known as `Tee`.

> The term 'Tee' is named after the Unix command 'tee', which is named after the
> T-shaped pipe fitting.

```csharp
public static T Tee<T>(this T @this, Action<T> action)
{
    if (@this is not null && @this is not None<T>)
        action(@this);
    return @this;
}
```

We can now achieve the same, but calling the `Tee` function where we want:
```csharp
public void TestTeeFunctional(Person person)
{
    var spouseFirstName = person
        .Map(a => a.Spouse)
        .Tee(d => _testOutputHelper.WriteLine(d switch
                    {
                    Some<Person> spouse  => $"{spouse.Value.FirstName} is {spouse.Value.Age} years old",
                    _ => "No spouse"
                    }))
    .Map(spouse => spouse.FirstName);

    Assert.Equal("Jane", spouseFirstName);
}
```

As you can see, the logic for printing whether the spouse has a value is now
grouped within the same scope, and not divided over multiple lines. After we're
done logging the spouse value, we can continue mapping as before. This allows us
to perform additional operations on the same result without breaking the
pipeline, making our code more readable and maintainable.

# Mutability

I really like the fact that the Rust programming language is designed to be
immutable by default, unless specified otherwise, which can be challenging for
some developers at first. However, it is a good practice to consider whether a
value should be mutable or not, as many issues with state and concurrency can
lead to various bugs. By defaulting to immutability, we force ourselves to think
about these issues more carefully.

In C#, the introduction of records and the `with` operator has made it much
easier to have immutable types and values. In our `Person` record, the fields are
immutable, so they can only be set once on an instance.

```csharp
private readonly Person _person;

public ImmutabilityTests()
{
    // Create a new instance of the Person class.
    // All properties are readonly, so we can't change the value of the properties.
    _person = new Person
    {
        FirstName = "John",
                  LastName = "Doe",
                  Age = 42,
    };
}

[Fact]
public void TestImmutablePerson()
{
    // We can't change the value of the FirstName property, because it is readonly.
    _person.FirstName = "Jane";
    // We can't change the value of the LastName property, because it is readonly.
    _person.LastName = "Doe";
    // We can't change the value of the Age property, because it is readonly.
     _person.Age = 42;
    // We can't change the value of the Spouse property, because it is readonly.
    _person.Spouse = new Person
    {
        FirstName = "Jane",
        LastName = "Doe",
        Age = 42
    };
    Assert.Equal("John", _person.FirstName);
}

```

Our compiler will not allow the code snippet above to compile, as the properties
on the `Person` record can only be set when the instance is instantiated.
Therefore, what we actually need is an updated version of that instance. The
`with` operator allows us to create a copy of the original record, initializing
the members with the same values as the original, or a different value if
specified within the `with` operator. Since the fields on our `Person` record
are immutable, this will create a new instance of the `Person` record with the
updated fields.

```csharp
[Fact]
public void TestImmutablePersonWithWith()
{
    var alteredPerson = _person.Map(p => p with { FirstName = "Jane" }).Unwrap();
    Assert.Equal("John", _person.FirstName);
    Assert.Equal("Jane", alteredPerson.FirstName);
}
```

If another thread comes along and alters the `_person` private field somewhere
within our sequence, our test would still succeed, as we're not referring to a
mutable instance. This is a prime example of pure functions, where we have no
side effects.

# References

If you want
to leverage this in your own project there are a number of packages you could
include in your project to get started quickly:

- [LanguageExt](https://github.com/louthy/language-ext)
- [RSharp](https://github.com/tluijken/rsharp) (written by yours truly)
- [CSharpFunctionalExtensions](https://github.com/vkhorikov/CSharpFunctionalExtensions)

Also a big shoutout to [Simon Painter](https://github.com/madSimonJ) for
providing excellent content via
[YouTube](https://www.youtube.com/results?search_query=simon+painter).

# Closing words

I hope you enjoyed this post on functional programming. By leveraging these
concepts, we can create more maintainable, reusable and testable code. Although
functional programming can be daunting at first, with a little bit of practice
and experimentation, it can be a very powerful tool in your development arsenal.
I hope you found this post informative and valuable.

Thank you for reading!

