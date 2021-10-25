+++
author = "Michael Boyles"
title = "Design patterns reviewed #3: behavioural"
date = "2021-10-25"
description = "A review of the behavioural design patterns used in software development"
tags = ["design"]
location = "Chester, England"
+++

This is the third and final post in a series in which I’m reviewing
[software design patterns](https://en.wikipedia.org/wiki/Software_design_pattern). This time out I'm taking a look at
[behavioural patterns](https://en.wikipedia.org/wiki/Behavioral_pattern). These all "*characterize the ways in which
classes or objects interact*". If that sounds indistinct from structural patterns, it's because it is. Just go with it.

<!--more-->

In alphabetical order, they are:

## [Chain of Responsibility](https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern)

In practice, a chain of responsibility often ends up looking like an instance of the decorator pattern, which is my
favourite pattern. For whatever reason, decorators are categorized as a structural pattern,
[previously covered here](/post/structural-design-patterns/#decoratorhttpsenwikipediaorgwikidecorator_pattern).
Some people draw a few minor distinctions between them, like that a decorator always delegates down the chain, while a
chain of responsibility can stop when a request is handled.

## [Command](https://en.wikipedia.org/wiki/Command_pattern)

Maybe the command pattern was more revolutionary in the past, or maybe it's just less relevant in Java, but I feel this
is too basic to qualify as a design pattern. It's really just the idea of passing functions as arguments. A generic
"Command" interface is no more useful than [`Runnable`](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html),
which has existed in Java since Java 1.0, and became easier to work with in Java 8 and the introduction of lambda
functions.

## [Interpreter](https://en.wikipedia.org/wiki/Interpreter_pattern)

The interpreter pattern is really just an application of the [composite pattern](/post/structural-design-patterns/#compositehttpsenwikipediaorgwikicomposite_pattern)
in a specific context: a language built from composable expressions. It's good but there doesn't need to be a
distinction.

## [Mediator](https://en.wikipedia.org/wiki/Mediator_pattern)

The core idea seems fine but the name and the implementation put me off. The book gives an example where two widgets
on a form affect one another, and that logic which governs that interaction is pulled into a "mediator" rather than placed
in one (or both) of the widgets. I think aiming to decouple the widgets from their interactions is good. You can achieve
something similar with observers though.

## [Memento](https://en.wikipedia.org/wiki/Memento_pattern)

This pattern aims to capture a mutable object's state so that it can be restored later. Immutability is often better.
By keeping a stack of immutable objects, you can restore state by popping off elements.

## [Null Object](https://en.wikipedia.org/wiki/Null_object_pattern)

The null object pattern is an excellent alternative to returning a null reference. Not every interface lends itself to
having a null implementation though. A method which returns a primitive might have no sensible return value.

## [Observer](https://en.wikipedia.org/wiki/Observer_pattern)

In moderation, this one's simple but effective. In a complex application, especially ones where these relationships
between classes are managed automatically (e.g. a dependency injection framework), you can quickly run into circular
dependencies. An effective way to solve this is to introduce a broker between the source and the target; this way, every
component depends on the broker and not directly on other logical components.

## [Specification](https://en.wikipedia.org/wiki/Specification_pattern)

The specification pattern is the same as the interpreter pattern. Ultimately, what you're doing is composing expressions.

## [State](https://en.wikipedia.org/wiki/State_pattern)

The state pattern is conceptually similar to the proxy pattern The only real difference is that a proxy has a single,
fixed delegate and the state pattern will mutably adjust its delegate according to some state transitions.

Terrible name, by the way. Almost any object is going to include state.

## [Strategy](https://en.wikipedia.org/wiki/Strategy_pattern)

This is a great one. It's basically an object-oriented application of a functional idea.
[Java's Comparator](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Collections.html#sort(java.util.List,java.util.Comparator))
is a nice example. There are infinitely many ways you might want to sort a collection of items and the majority of the
sorting logic can be implemented generically. As a user you just need to plug in a function which compares two items.

## [Template method](https://en.wikipedia.org/wiki/Template_method_pattern)

The template method is an ugly cousin to the strategy pattern. It's worse because it relies on inheritance. You probably
shouldn't actively look for reasons to use it, but it's not going to make a class hierarchy that already uses
inheritance any worse.

## [Visitor](https://en.wikipedia.org/wiki/Visitor_pattern)

The visitor pattern is great for processing an abstract syntax tree and... I think that's about it. I've never seen it
used outside that context, except for contrived examples involving domestic animals.

---

[Part 1 - Creational Design Patterns](/post/creational-design-patterns)

[Part 2 - Structural Design Patterns](/post/structural-design-patterns)
