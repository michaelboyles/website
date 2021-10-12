+++
author = "Michael Boyles"
title = "Design patterns reviewed - part 1: creational"
date = "2020-08-25"
description = "A review of the creational design patterns used in software development"
tags = ["design"]
+++

This is the first post in a series in which I’ll be reviewing [software design patterns](https://en.wikipedia.org/wiki/Software_design_pattern), giving my opinions on which you should use and which you should avoid. I've grouped them into the same categories as they are in the Gang of Four’s [*Design Patterns*](https://www.amazon.co.uk/Design-patterns-elements-reusable-object-oriented/dp/0201633612). The first group I’ll be covering are [creational patterns](https://en.wikipedia.org/wiki/Creational_pattern). These are all strategies that you can choose when deciding how to instantiate an object.

<!--more-->

In alphabetical order, they are:

## [Abstract factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern)

This one’s pretty good but in practice I find that “families of products”, as the book calls them, aren’t that common.

I don’t think the name is very good; I would have called it a ‘group factory'. It also feels more like a specialization of the factory method pattern than its own distinct thing; the book already defines a few variations on that pattern so I don’t see how one more would have hurt.

## [Builder](https://en.wikipedia.org/wiki/Builder_pattern)

This pattern’s okay but use it sparingly. It makes it easy to create big immutable objects, without having to resort to a long, unreadable constructor. If your class has so many fields that it starts to require a builder then you should really consider whether the class can be broken up before choosing this pattern. I mostly use it for creating objects which represent JSON or XML.

One problem with traditional builder implementations is that it’s easy to miss fields, resulting in `null` field values, and problems that can only be caught at runtime. *Immutables* has a feature called [staged builders](https://immutables.github.io/immutable.html#staged-builder) which ensures that all fields are populated at compile-time, at the cost of creating a lot of garbage at runtime.

I see this pattern as basically just a workaround for a language not having [named parameters](https://en.wikipedia.org/wiki/Named_parameter) as one of its features.

If you’re going to use this pattern, don’t write the boilerplate yourself. Use [Lombok’s `@Builder`](https://projectlombok.org/features/Builder) or [Immutables](https://immutables.github.io/).

## [Factory method](https://en.wikipedia.org/wiki/Factory_method_pattern)

This one’s okay. A lot of examples I found for this pattern show a factory containing conditional statements comparing Strings or enums to select the concrete type. The book actually calls this variation a ‘*parameterized* factory method’:

```java
public Foo makeFoo(String fooType) {
    if (fooType.equals("BAR") {
        return new FooBar();
    }
    else if (fooType.equals("BAZ")) {
        Return new FooBaz();
    }
    return null; // or throw exception
}
```
I think this is usually a bad idea because you’ll run into problems at runtime if the method doesn’t comprehensively cover all possibilities, and it’ll become a pain to maintain.

On the other hand, if you just want to abstract away a concrete class then using the normal version of this partern is a fine solution. As a rule of thumb I’d say that if the arguments to the factory method are used for any other purpose than as arguments to the constructors of the concrete types then you’re doing something wrong.

## [Lazy Initialization](https://en.wikipedia.org/wiki/Lazy_initialization)

I’m not really convinced that this is a pattern in its own right but I think the idea is sound.

I’d usually achieve a similar result with a [memoizing](https://en.wikipedia.org/wiki/Memoization) [decorator](https://en.wikipedia.org/wiki/Decorator_pattern). The decorated class is responsible for performing the calculation and will perform it each time, even if it’s slow. The decorator is responsible for implementing a strategy to optimize the number of calls to the delegate, by saving the first result, for example.

## [Object Pool](https://en.wikipedia.org/wiki/Object_pool_pattern)

This is a good one in very specific situations when optimizing performance, like Java’s `Integer` constant pool.

Only use it when you’ve profiled your code and have identified a need for it, otherwise you’ll make your design more complex and [probably only make the performance worse](https://www.ibm.com/developerworks/java/library/j-jtp01274/index.html?ca=drs-).

## [Prototype](https://en.wikipedia.org/wiki/Prototype_pattern)

In order for this pattern to be useful, it seems to require that objects be mutable. There’s not much reason to clone an immutable object when you could just use the original directly, so I feel most usages of this pattern are going to involve a clone-then-mutate:

```java
Foo foo = getFoo();
Foo cloned = foo.clone();
cloned.setBar("new value");
```

I think that most objects should be immutable, so I’ve never found a use for this pattern. A better approach would be a [`toBuilder` instance method](https://projectlombok.org/features/Builder), which creates a new mutable builder using the source object’s fields. You can mutate the builder instance as much as you like and then use it to build a new, immutable, partial copy of the original.

## [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern)

Probably the worst pattern. A singleton is a form of global state. A dependency on one makes classes hard to test because you can’t easily mock or stub its methods. Don’t use it. Rely on dependency injection instead.

---

[Part 2 - Structural Design Patterns](/post/structural-design-patterns)

