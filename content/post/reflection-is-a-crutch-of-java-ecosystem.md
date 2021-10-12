+++
author = "Michael Boyles"
title = "Reflection is a crutch of the Java ecosystem"
date = "2021-10-04"
description = "Java libraries have an over-reliance on reflection as a means to solving problems generically. In this post I explain how code generation might be a better solution."
tags = ["design"]
location = "Chester, England"
+++

Java gained traction because you could write code which could run anywhere, but it continues to be successful in a large
part due to its fantastic ecosystem. Libraries like Spring, Hibernate and Gson are comparable to, if not better than,
practically any equivalent tools available for other languages. However, they've helped popularise the idea that using
[reflection](https://www.oracle.com/technical-resources/articles/java/javareflection.html) is likely to be an intrinsic
part of writing a Java library or framework. As Java developers, we're being short-changed with solutions that force us
to sacrifice strong typing and are impossible to debug.

<!--more-->

For each of the popular libraries I've already mentioned — Spring, Hibernate and Gson — I'll explain how an alternative
implementation could remove the need for reflection and potentially result in more performant, less error-prone code.
The examples are deliberately simple and are only intended to be illustrative of the general idea.

## [Gson](https://github.com/google/gson)

A basic case of deserializing some JSON with Gson looks like this:

```java
Book book = new Gson().fromJson(
    "{\"pages\": 123}",
    Book.class
);
```

Gson uses reflection to check what fields, methods and constructors are present in the provided class. It will create a
new instance and perhaps invoke a setter for each property, or even access the fields directly.

The analysis may be somewhat slow (relative to other operations, at least) for the first invocation for each class but
I expect that it will cache the result. A bigger problem is that `fromJson` necessarily has to accept any `Class`
instance as an argument. I could pass `StringBuilder.class`, or `InputStream.class`, or any other garbage. You can't
necessarily tell by looking whether the argument represents a JSON object or not and it might fail at runtime.

Instead, Gson could generate a serializer and deserializer pair for each JSON class:

```java
@Generated
class BookDeserializer
   implements JsonDeserializer<Book>
{
   Book fromJson(String json) {
       // jsonToTree from e.g. a parent class
       JsonNode root = jsonToTree(json);
       return new Book(
           root.getInt("pages")
       );
   }
}

// Sample usage
Book book = new BookDeserializer()
    .fromJson("{\"pages\": 123}"); 
```

A `BookDeserializer` now only accepts `Book`s. There will be no such thing as a `StringBuilderDeserializer` so there's
no possibility I can try to create a `StringBuilder` from some JSON.

## [Hibernate](https://hibernate.org/orm/)

Hibernate is an [ORM framework](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) which attempts to
bridge the gap between an object-oriented application and the representation of those objects in a relational database.
Hibernate operates on what it calls “entities”: in the simple case, a class would model a database table and an instance
of that class would model a row.

Hibernate suffers from the same issues as Gson but, on top of that, its implementation makes it extremely tedious to
debug. Because Hibernate has to be flexible enough to accept any possible input, what may seem like a simple operation —
say, to insert a row — can balloon in complexity. There are likely many code paths which your application will never
take and could be completely omitted in a generated implementation.

Generating an ORM implementation lets you lift the fundamental operations like constructing a SQL statement from deep
within the library to a much higher level.

```java
@Generated
class HibernateDB {
   // constructor

   BooksTable books() {
       return new BooksTable(entityManager);
   }
}

@Generated
class BooksTable implements HibernateTable<Book> {
   private final EntityManager entityManager;

   // constructor

   void insert(Book book) {
       entityManager.createNativeQuery("INSERT INTO person (id, pages) VALUES (?,?)")
           .setParameter(1, book.getId())
           .setParameter(2, book.getPages())
           .executeUpdate();
   }
}

// Sample usage
new HibernateDB(SqlDialect.ORACLE, "localhost", 1234)
   .books()
   .insert(new Book(123));
```

## [Spring](https://spring.io/)

Spring's dependency injection happens when your application starts. It builds the dependency graph, determines the order
in which components must be created, and wires everything together at runtime. There are plenty of potential issues that 
you can't catch until you spin up your application, like a missing or cyclic dependency.

In a world running on Kubernetes or serverless architectures where cold starts are common, the time spent building this
graph can be significant. I've written modular applications which relied on Spring's dependency injection to dynamically
pick up plugins, but most applications don't require that functionality; most applications have the same dependency
graph every time they run. A better approach for most people would be to analyze the dependency graph at compile-time
and generate a class which acts as the glue.

```java
@Generated
class MyDI {
    public App buildApp() {
        final ComponentA aaa = new ComponentA();
        final ComponentB bbb = new ComponentB(aaa);
        final ComponentC ccc = new ComponentC(aaa, bbb);
        return new App(aaa, bbb, ccc);
    }
}
```

Of course, Spring does much more than just dependency injection but [Micronaut](https://micronaut.io/)'s authors are
achieving a huge number of the same features without using reflection, often through similar generative methods.

## The case for code generation

One of the best things about generating code is that it lets us think about code structure in a completely different
way. We usually feel uneasy about duplicating similar logic all over our codebase. It's a maintenance nightmare. At a
certain point, we tend towards more generic solutions but generic solutions can be confusing unless you understand the
full context of the problem. A generative approach lets us completely flip that on its head: how would you write your
application differently if none of the disadvantages of duplication existed?

Of course, the complexity of creating a generic solution for a complex problem will never just disappear. All generating
code does is shift the responsibility of supporting multiple use-cases from the runtime library to a new generator
component e.g. a Maven plugin. Thanks to libraries like [JavaPoet](https://github.com/square/javapoet), that needn't be
significantly more effort for a library author either, and the advantages can be huge. An interstitial, human-readable
output gives users greater transparency about what a library is really doing and a more accessible entry-point for
solving their own problems.
