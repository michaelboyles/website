+++
author = "Michael Boyles"
title = "SOLID: the 'SO' stands for Snake Oil"
date = "2020-09-13"
description = "SOLID is possibly the most over-hyped concept in all of software engineering. In this post I explain why."
tags = ["design"]
+++

SOLID is possibly the most over-hyped concept in all of software engineering. You can throw away that jar of Sudocrem
because we'll have no fence-sitting today, no sir.

<!--more-->

I often hear SOLID touted as some profound programming philosophy with the power to absolve us from all of our software
sins. Like a Buddhist achieving enlightenment, it is a doctrine that requires deep and profound meditation but, once
internalized, you will unlock [a truly limitless potential](https://stackoverflow.com/users/22656/jon-skeet).

It may be comforting to convince yourself that that's true but it's not. I think that software engineers sometimes feel
like imposters within engineering. Civil engineers don’t build bridges knowing there are cracks in the foundations that
they can just patch after it’s open (“don’t worry, guys, we’ve got a Jira for that”). Well,
[maybe they do](https://en.wikipedia.org/wiki/List_of_bridge_failures) but the point is that these insecurities bring
along with them a desire to identify some fundamental principles of our discipline, so that we can tell ourselves that
we're definitely not just [monkey's sitting at typewriters](https://en.wikipedia.org/wiki/Infinite_monkey_theorem). No,
we’re applying established axioms of design.

I believe that SOLID completely fails in its attempt to help us to design and write better software; it’s perfectly
possible to follow all of the principles to the letter and still produce some absolute garbage. They’re simultaneously
banal enough that any experienced developer should already be following them, and obtuse enough to confuse and disorient
a beginner. Saying something in a convoluted way does not make the underlying idea deep or complex.

Let's break them down one by one and see what we’re dealing with.

## [Single-Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle)

This principle states a class should have one responsibility or "*only one reason to change*". The natural follow-up
question is “what constitutes a 'responsibility'?” It’s often
[debatable](https://softwareengineering.stackexchange.com/questions/345018/when-using-the-single-responsibility-principle-what-constitutes-a-responsibili).

The underlying idea here is sound but it's hardly earth-shattering. I’d word it another way: keep your classes as small
as possible. Smaller classes are easier to understand, are more maintainable, and are easier to test. My advice is more
direct but of course it has the problem: when *precisely* should you start to break out the functionality of one class
into multiple classes? We’ve all been unlucky enough to see a few monstrous 2000-line “classes” which have clearly
crossed that line but there’s a fair bit of grey area too. You’ll get a better sense of where that line sits through
experience, not by reading this mumbo-jumbo.

## [Open-Closed Principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) 

This is by far the most offensively awful tenet of SOLID, and often cited as being
["the most important"](https://www.dictionary.com/browse/wrong) by Uncle Bob[^1]. It states "*software entities
(classes, modules, functions, etc.) should be open for extension, but closed for modification*". What this advocates is
that you should extend a class and perform your bug fix or change in behaviour by way of overriding its methods, rather
than by altering it existing implementation.

Inheritance is usually awful. It’s very often the wrong approach and should be used very sparingly. Classes with complex
inheritance hierarchies are hard to understand and hard to test exhaustively. You should not be looking for reasons to
use it proactively.

This principle was [redefined to apply to interfaces](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle#Polymorphic_open%E2%80%93closed_principle).
Effectively it states that your interfaces should not change, but that you may add new implementations.

If anything speaks to SOLID’s lack of coherency, it’s that they had to retroactively completely redefine one of the
definitions.

Both variants of this principle are extremely averse to change, which I don't understand. Making breaking changes for
clients should not be done lightly but sometimes we have to do that for the good of the overall system design. As long
as those changes are properly communicated ([semver](https://semver.org/), a migration guide, etc.), with adequate time
to transition then it shouldn't be a huge problem. Software development is often an iterative process. We learn more
about the optimal design as we build the system; that's what makes Agile such a popular methodology. This principle
seems to directly contradict that idea.

We should build systems which are flexible to change by striving to choose the right abstractions and loosely coupling
components, but if we later learn that we choose the wrong abstractions, that shouldn’t be a lifelong burden that we
need to live with.

## [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

In essence, what this principle states is that if a method is declared as accepting a Collection and it’s passed an
ArrayList then it should work as expected (explained below if you’re not Java-savvy).

When implementing or extending a type then you should be careful not to break the contracts they’ve defined. Of course
that’s true but that’s a fundamental aspect of subtyping. Maybe this was a revolutionary idea in the 1980s but are there 
still people who are oblivious to this, excluding complete beginners? If something this basic made the list then why
didn’t any number of other basic core ideas like encapsulation?

## [Interface segregation principle](https://en.wikipedia.org/wiki/Interface_segregation_principle)

This principle kind of encompasses two things. The first is to design your interfaces so that they are small and
specialized ("segregation"). Java's collections framework does this quite well. For example this hierarchy, from most
generic to most specific:

 - Iterable: something you can iterate over
 - Collection: something you can add and remove elements from
 - List: a sequence supporting random access
 - ArrayList: an array-based implementation

I agree with this but designing interfaces - choosing the right abstractions - is hard. It’s something you can only
learn from experience. You can only learn *how* to follow this principle by trial and error. I doubt that anyone had
been deluding themselves into thinking that their interface with 100 methods was a stroke of genius until this came
along.
 
This principle goes on to advocate that you give precedence to the most-general abstraction: "*no client should be
forced to depend on methods it does not use*", which is of course aided by having specialized interfaces. Nothing wrong
with that.

## [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle)

This principle states "*abstractions should not depend on details*". Effectively what this says is that if your method
only requires the methods of Collection, don't force it to accept an ArrayList. The existence of an array in that case
is and should remain an implementation detail.

Once again I agree, but there is nothing particularly profound to be learned here that
[loose coupling](https://en.wikipedia.org/wiki/Loose_coupling) doesn't already express in a more concise, coherent way.

---

There we have it. Four inoffensive but uninspiring ideas sitting alongside one outright terrible one, as though they
have equal merit.

I don’t believe you should trash something unless you’re able to propose a better alternative. Here are what I believe
to be the essential concepts when it comes to object-oriented design:

 - Encapsulation
 - Contemplate the right abstractions
 - Immutability
 - Avoid null references
 - Shorts methods and classes

Of course there are probably hundreds more metrics by which you could judge whether some software is well-written -
meaningful identifiers, tests, etc. - but those are applicable to other paradigms too.

Now all I need to do is reword them into a nice acrostic and I’ll be a millionaire too.

[^1]: I couldn't find any direct source for this but this attribution is all over the internet, rightly or wrongly
