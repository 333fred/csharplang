# Factory Constructors (v2)

## Background

This proposal builds off of @jaredpar's proposal, found [here](https://github.com/jaredpar/csharplang/blob/factory/proposals/factory-methods.md). This
proposal, however, goes in a slightly different tack. Many of the issues the LDM found with the original proposal were around the sheer broadness of it.
The v1 proposal attempts to allow any arbitrary method to be marked as a constructor (with the implementation subject to stricter rules), and the result
is that there are a few issues:

* Object initializers would be basically allowed (syntactically) after any method call. This introduces parsing issues with other areas the LDM has
expressed interest in, such as block expressions, that would be either unsolvable or require a significant amount of lookahead to resolve ambiguities.
This would result in poor IDE experiences around incomplete code, which several members of the LDM were concerned about.
* As part of this being allowed after every method call, several members of the LDM suggested that we could then allow object initializers after any
expression, and allow setting of properties that can be set in that location (ie that have accessible `set` methods) to be set inside. Certainly, the
syntactic form suggested in the v1 proposal leads one to assume that initializers can go anywhere: from the consumption side, there is no special
syntax required to call the `[Factory]` method. Additionally, this would bring a symmetry with the VB `With` statement, which allows the receiver
of a bunch of property accesses to be omitted. However, another segment of the LDM felt that this was exposing mutation from object initializers,
which don't expose mutation today. Initializers can be viewed as occurring during the construction phase of the object: indeed, the `init` properties
proposal takes advantage of this fact explicitly to allow a new scope for setting properties. By allowing initializers to mutate existing object that
have already been observed previously, this segment of the LDM felt that we would be diluting the meaning of object initializers, and that users
would have an additional thing to look at in their heads when reading an expression.

This new proposal, Factories (v2), attempts address these concerns by limiting valid syntactic locations for object initializers and introducing a
new syntactic form for factory constructors. This plays well into the ability for record types to be purely syntactic sugar, and has a clear delineation
of what would be necessary for a C# 9 release of records while allowing us to continue iterating in C# 10 and beyond.

## Motivation

The impetus for this proposal (and the v1 proposal) is of course the need for a general purpose strategy for `with` expressions in C# with the
addition of record types. Generally, a `with` expression needs a method that is guaranteed to return a fresh copy of an existing object that has not
yet been observed (outside of whatever a user does in their constructor). This method must be `virtual`, as the value returned should be of the same
runtime time as the value the `with` expression was called on, which may be more derived than the compile-time type of the expression. However, there
are some more general motivating patterns that .NET developers use today which can help inform design decisions here:

* Factories: Not factories as defined in this proposal, but the Factory pattern as implemented by many, many DI frameworks. Users use factories today
in order to decouple their implementations from their interfaces for many different reasons, including testability, ability to swap strategies as
needed, as a public extension point, and more. These factories tend to consist of anything from a series of static methods in simple cases to full
classes where the factory itself is injected with many instance methods for constructing things.
* Static `Create` methods: These methods are often introduced as a substitute for generic inference in constructors. In C# today, there is no generic
inference for constructor type parameters: they must be fully spelled out. There are a few different proposals for partial inference or full inference
today, but none have been implemented.

This proposal does not attempt to solve the second bullet here, but is intended to be delivered at the same time a solution to that problem as a
general construction improvement iteration to C#.

## Detailed Design

We introduce a new type of constructor: a factory constructor. These constructors do not have to be defined in the same class that they create, can
be virtual, can be static, and can return more derived instances of the type they construct. These factory constructors have restrictions on what they
can return, much like the v1 proposal has, in order to ensure that the instance returned is unobservered outside of the factory call.

```cs
public class Widget
{
    public string Name { get; set; }
    public int Id { get; init; }

    internal Widget() { }
}

public static class WidgetFactory
{
    public static factory Widget() => new Widget();
}

var w = new WidgetFactory.Widget()
{
    Name = "333fred",
    Id = 43
};
```

### Grammar

Factory constructors are specified by the following grammar:

```antlr
factory_constructor_declaration
    : attributes? factory_constructor_modifier* 'factory' factory_constructor_declarator factory_constructor_body
    ;

factory_constructor_modifier
    : 'new'
    | 'public'
    | 'protected'
    | 'internal'
    | 'private'
    | 'static'
    | 'virtual'
    | 'sealed'
    | 'override'
    | 'abstract'
    | factory_constructor_modifier_unsafe
    ;

factory_constructor_modifier_unsafe
    : 'unsafe'
    ;

factory_constructor_declarator
    : type '(' formal_parameter_list? ')'
    ;

factory_constructor_body
    : block
    | ';'
    ;
```

Some examples:

```cs
public static interface I
{
    int MyProperty { get; init; }
    // Define a copy constructor for the ability to `with` any `I` implementation
    factory I();
}
public abstract class AbsI : I
{
    public int MyProperty { get; init; }

    protected abstract new factory I();
    factory I.I()
        => new this.I() // `this.I` is required to disabmiguate from the default parameterless constructor
           {
                MyField = this.MyField
           };
}
public abstract class DerivedI : AbsI
{
    public int DerivedProperty { get; set; }
    protected override factory I() => new DerivedProperty() { DerivedProperty = this.DerivedProperty };
}

public class IFactory
{
    public factory I(int param1, int param2) => ...;
    public factory I(string param1) => ...;
    public static factory I(object param1) => ...;
}
class Widget
{
    static factory Widget() => new Widget(string.Empty, -1);
    static factory Widget(Widget widget) => new Widget() { Title = widget.Title, Id = widget.Id };
    protected Widget(string title, int id) => (Title, Id) = (title, id);
    public string Title { get; init; }
    public int Id { get; set; }
}
```

Factory constructors are invoked by calling `new`, and providing a primary expression that resolves to a factory constructor.

```antlr
object_creation_expression
    : 'new' primary_expression_or_type '(' argument_list? ')' object_or_collection_initializer?
    | 'new' primary_expression_or_type object_or_collection_initializer
    ;

primary_expression_or_type
    : primary_expression
    | type
    ;
```

For an object creation expression, the compiler will first attempt to bind the `primary_expression_or_type` as a type. If it binds,
and if overload resolution with the provided arguments returns anything other than an empty set, then no attempt is made to bind
the `primary_expression_or_type` as a `primary_expression`. However, if it fails to bind as a type or overload resolution fails to
find an applicable constructor, the compiler will bind the `primary_expression_or_type` as a `primary_expression`, and attempt to
resolve it as a method group. If overload resolution returns a factory constructor, then that is the constructor that is used by the
object creation expression.

Examples:

```cs
var iFactory = new IFactory();
var i1 = new iFactory.I(1, 2) { MyProperty = 3 };
var i2 = new iFactory.I("hello world!");
var i3 = new (new IFactory.I(null)).I();
```

Factory constructors have the following restrictions:

* The return expression must be
  * new expression including object / collection initializers
  * Call to a member annotated with [Factory]
  * A struct value
  * A ternary or collascing expression where both branches meet the above requirements
* Cannot be used outside of an object creation expression.

### Metadata Encoding

Factory constructors are encoded as standard methods with an unspeakable name and the SpecialName flag.

## Questions

### Should instance factory constructors conflict with existing constructors of the same type?

Basically, if a type has a constructor with parameters (int, int), should that same type be able to define a factory constructor
with the same parameters? If we want to make a parameterless "give me a copy" instance factory constructor, as in the examples above,
then this will be required, and we'll need to have a disambiguation syntax (`this.`, for example).

### Use of the identifier `factory`

This proposal introduces a new contextual keyword, `factory`, and if a user had a type named lowercase `factory` they would either
be unable to use the feature, or upgrading to C# (whatever version this would ship in) would break them. Is there an existing keyword
or different syntax we could use to avoid this?

### Allow `async` factories

Another big reason people create static factory methods to construct objects is in order to make them async, which constructors today
do not allow. We could conceivably allow specifying the `async` keyword in the factory signature and have the factory return a `Task<T>`,
or any other task-like type.

## Considerations

### Splitting up the feature

The full implementation of this proposal would likely not fit in the initial version of records, so we necessarily need to look
at what parts of the feature could be realistically pushed back to C# 10 or later. There is a nice dividing line here: we can scope
the feature down as far as only permitting parameter-less instance factories, and only those factories that construct a type that the
current type inherits from. We could further scope it down by not permitting these copy constructors to be invoked from C# code except
by a `with` expression. This would permit the LDM's stated goal of having records be only a syntax sugar that can be converted to and
from transparently to consumers, while not blocking off the ability to bring a more general feature here in the future.
