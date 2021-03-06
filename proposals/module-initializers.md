# Module Initializers

* [x] Proposed
* [ ] Prototype: [In Progress](https://github.com/jnm2/roslyn/tree/module_initializer)
* [ ] Implementation: [In Progress](https://github.com/dotnet/roslyn/tree/features/module-initializers)
* [ ] Specification: [Not Started]()

## Summary
[summary]: #summary

Although the .NET platform has a feature that directly supports writing initialization code for the assembly (technically, the module), it is not exposed in C#.  This is a rather niche scenario, but once you run into it the solutions appear to be pretty painful.  There are reports of [a number of customers](https://www.google.com/search?q=.net+module+constructor+c%23&oq=.net+module+constructor) (inside and outside Microsoft) struggling with the problem, and there are no doubt more undocumented cases.

## Motivation
[motivation]: #motivation

- Enable libraries to do eager, one-time initialization when loaded, with minimal overhead and without the user needing to explicitly call anything
- One particular pain point of current `static` constructor approaches is that the runtime must do additional checks on usage of a type with a static constructor, in order to decide whether the static constructor needs to be run or not. This adds measurable overhead.
- Enable source generators to run some global initialization logic without the user needing to explicitly call anything

## Detailed design
[design]: #detailed-design

The C# compiler will recognize the following attribute:

``` c#
namespace System.Runtime.CompilerServices
{
    // Note: an Obsolete attribute is not needed here,
    // because only C# 9 compilers will have access to attributes added in .NET 5.
    // LangVersion checks will still be necessary.
    [AttributeUsage(AttributeTargets.Module, AllowMultiple = false)]
    public class ModuleInitializerAttribute : Attribute
    {
        public ModuleInitializerAttribute(Type type) { }
    }
}
```

You would use it like this

``` c#
using System.Runtime.CompilerServices;

[module: ModuleInitializer(typeof(MyModuleInitializer))]

internal static class MyModuleInitializer
{
    static MyModuleInitializer()
    {
        // put your module initializer here
    }
}
```

and the C# compiler would then emit a module constructor that causes the static constructor of the identified type to be triggered:

``` c#
void .cctor()
{
    // synthesize and call a dummy method with an unspeakable name,
    // which will cause the runtime to call the static constructor
    MyModuleInitializer.<TriggerClassConstructor>();
}
```

Note that the overhead of checking whether the static constructor has run on a module initializer type (`MyModuleInitializer` in this example) can be kept to a minimum by referencing the type only in the `[module: ModuleInitializer(typeof(MyModuleInitializer))]` attribute.

This proposal uses `AttributeUsage(AllowMultiple = false)` to disallow the user from declaring more than one initializer class without adding any special language rules to accomplish that. It is very lightweight in that it uses existing syntax and semantic rules for attributes. On the other hand, it may make it more difficult for users to realize that the static constructor they are looking at is the module initializer, because the attribute may be far away from the class that actually contains the module initializer, and it also requires us to decide what happens if the attribute is used to reference a type from another assembly.

## Unresolved questions
[unresolved]: #unresolved-questions

#### Should we permit multiple types to be decorated with ModuleInitializerAttribute in a compilation? If so, in what order should the static constructors be invoked?

One option: do the same thing we do for static initializers in partial classes. Order them based on file order+source position.

#### Should we permit using a type from another assembly as the module initializer?

e.g.  `[module: ModuleInitializerAttribute(typeof(InitializerFromOtherAssembly))]`

This could make it difficult to invoke the class constructor simply by calling a dummy method, because we won't be able to synthesize such a method on the type. We could consider falling back to `System.Runtime.CompilerServices.RuntimeHelpers.RunClassConstructor()` in this case, but it introduces a dependency on the reflection stack.

This use case implies that the initializer class is not the module initializer for the assembly it is declared in. It feels like this use case should instead by handled by exposing a method that the external consumer should call from their module initializer.

```cs
// Assembly 1
public class MyLibInit
{
    public static void Init() { }
}

// Assembly 2
using System.Runtime.CompilerServices;

[module: ModuleInitializer(typeof(MyInit))]

class MyInit
{
    static
    {
        MyLibInit.Init();
    }
}
```

## Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Perhaps the existing third-party tooling for "injecting" module initializers is sufficient for users who have been asking for this feature.

## Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

There are a number of possible ways of exposing this feature in the language:

### 1. Special global method declaration

A module initializer would be provided by writing a special kind of method in the global scope:

```csharp
internal void operator init() ...
```

This gives the new language construct its own syntax. However, given how rare and niche the scenario is, this is probably far too heavyweight an approach.

### 2. Attribute on the type to be initialized

Instead of a module-level attribute, perhaps the attribute would be placed on the type to be initialized

```csharp
[ModuleInitializer]
class ToInitialize
{
    static ToInitialize() ...
}
```

With this approach, we would either need to reject a program that contains more than one application of this attribute, or provide some policy to define the ordering in case it is used multiple times. Either way, it is more complex than the original proposal above.

#### 3. Attribute on the static constructor to be initialized

Instead of a module-level attribute, perhaps the attribute would be placed on the static constructor to be initialized

```csharp
class ToInitialize
{
    [ModuleInitializer]
    static ToInitialize() ...
}
```

With this approach, we would either need to reject a program that contains more than one application of this attribute, or provide some policy to define the ordering in case it is used multiple times. Either way, it is more complex than the original proposal above.

#### 4. Attribute on a static method to be called

Instead of a module-level attribute, perhaps the attribute would be placed on the method to be called to perform the initialization

```csharp
class Any
{
    [ModuleInitializer]
    static void Initializer() ...
}
```

As in the previous approach, we would either need to reject a program that contains more than one application of this attribute, or provide some policy to define the ordering in case it is used multiple times. Either way, it is more complex than the original proposal. We also would probably need to reject any method with this attribute if it has parameters or a non-`void` return type.

#### 5. Do nothing
If we do not implement this feature, then:
- Users who really need module initializers continue to rely on third-party tooling to inject them into their assemblies.
- Source generators will have to rely on static constructors which add overhead or on Init() methods that users must explicitly call.

## Design meetings

### [April 8th, 2020](/meetings/2020/LDM-2020-04-08.md#module-initializers)
Let's let any static method be a module initializer (option 4 in the above proposal), and mark that method using a well-known attribute. We'll also allow multiple module initializer methods, and they will each be called in a reserved, but deterministic order.
