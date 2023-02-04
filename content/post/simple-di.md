+++
author = "Michael Boyles"
title = "A simple, compile-time dependency injection framework"
date = "2023-02-04"
description = "A high-level overview of a dependency injection framework I've written using code generation at compile-time"
tags = ["project", "java"]
location = "London, England"
+++

The most popular [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) frameworks in Java &mdash; 
Spring and Guice &mdash; are built using reflection. There are some problems with this approach, and I believe that a
compile-time solution is better in most cases. There are already compile-time implementations (e.g. 
[Dagger](https://github.com/google/dagger)), but I wanted to write
[my own bare-bones implementation](https://github.com/michaelboyles/simple-di). It's not designed to be used in real
projects, but as an illustrative example of how such an implementation can work.

<!--more-->

I see three flaws of using reflection for dependency injection:

  1. It's hard to debug if something's not wiring together as you expect. Stepping through code that has a heavy reliance
     on reflection is usually horrendous. Spring is great until you see how the sausage gets made.
  2. You don't catch problems until you actually run your application. The feedback loop for discovering problems is
     slower than it needs to be.
  3. There's a start-up cost. This might be negligible in a lot of cases, but you're doing some computation every 
     single time your application starts which usually only needs to be done once. Especially in cloud-based apps,
     optimizing start-up time can be important.

The reflection-based approach has its own benefits too. It means you can have runtime discovery of beans, and I've used
that to create plugin-like architectures. Most applications don't need that, though.

The alternative to reflection is to generate some code at compile-time which performs the wiring. That generated code
might look something like this:

```java
public final class SimpleDIContext {
    private final Map<String, Object> nameToBean = new HashMap<>();

    public SimpleDIContext() {
        PassengerSeat passengerSeat = new PassengerSeat();
        Driver driver = new Driver();
        DriversSeat driversSeat = new DriversSeat();
        Turbocharger turbocharger = new Turbocharger();
        Engine engine = new Engine(turbocharger);
        Car car = new Car(engine, List.of(driversSeat, passengerSeat));
        car.addDriver(driver);
        car.addSeats(new Seat[] {passengerSeat, driversSeat});
        nameToBean.put("passengerSeat", passengerSeat);
        nameToBean.put("driver", driver);
        nameToBean.put("driversSeat", driversSeat);
        nameToBean.put("turbocharger", turbocharger);
        nameToBean.put("engine", engine);
        nameToBean.put("car", car);
    }

    public Object getBeanByName(String name) {
        return nameToBean.get(name);
    }
}
```

In practice, this involves writing a build plugin which hooks in before the compilation phase. I used Google's
[AutoService](https://github.com/google/auto/tree/main/service), which means that all users need to do is to add a
`provided`-scope dependency to their project, [similar to Lombok](https://projectlombok.org/setup/maven).
I used [JavaPoet](https://github.com/square/javapoet) to generate Java source without worrying about tedious and
error-prone String manipulations.

To mark classes as beans, I chose
[`javax.inject`](https://docs.oracle.com/javaee/6/api/javax/inject/package-summary.html) as it already defines all the
basic things we'll need. By providing an implementation of [`AbstractProcessor`](https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/AbstractProcessor.html),
we can discover and process every class annotated with [`@Singleton`](https://docs.oracle.com/javaee/6/api/javax/inject/Singleton.html).

After finding all annotated classes, we need to work out how they should wire together, 
and in which order they should be created. You can't create `Car` without an `Engine`, and you can't create an `Engine`
without a `Turbocharger`, but how do you express that programmatically?

I noticed that as long as you construct everything in order of their total number of dependencies &mdash; both direct
and transitive &mdash; then the precise order you construct things in doesn't matter. If you first construct everything
with zero dependencies, then everything with precisely 1 dependency, then 2, and so on, then the exact permutation
doesn't matter. Two beans with the same total number of dependencies can't depend on each other.

All we need is a method which finds the total number of dependencies for each bean, then we can sort by that number.
This is a simple recursive problem. For each bean, the number of total dependencies is the number of direct dependencies,
plus the total number of dependencies for each of those. Once we've obtained that number for a given bean once, we can
store it in a map to memo it.

```java
private long getNumDependencies(Map<String, Long> fqnToNumDependents, Bean bean) {
    final Long SENTINEL = -123L;

    Long prevNumDeps = fqnToNumDependents.get(bean.getFqn());
    if (SENTINEL.equals(prevNumDeps)) throw new RuntimeException("Circular dependency!");
    if (prevNumDeps != null) return prevNumDeps;

    fqnToNumDependents.put(bean.getFqn(), SENTINEL);
    long numDependencies = 0;
    for (Bean dependency : bean.dependencies()) {
        numDependencies += (1 + getNumDependencies(fqnToNumDependents, dependency));
    }
    fqnToNumDependents.put(bean.getFqn(), numDependencies);
    return numDependencies;
}
```

The insertion and check for a sentinel is a simple protection against infinite recursion in the case of a circular
dependency. We insert the sentinel when we start computing the number for a bean, and if we come across it again then
it means we're going in circles. The sentinel is negative because it's not possible for something to have a negative
number of dependencies. A better implementation would keep track of the full path, so that users could see what was
causing the cycle.

There's a little more complexity for beans which depend on collections and
[`Provider`](https://docs.oracle.com/javaee/6/api/javax/inject/Provider.html)s, which I skipped above. Method injection
is easy; it can just be done after all construction injection.

I chose not to implement field injection. The Spring team
[recommend constructor injection](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-setter-injection)
anyway, and if a field is private then the only way to wire it is using reflection.

The full project is [hosted on GitHub](https://github.com/michaelboyles/simple-di). The readme explains how to build
and run it. There's a sample module included to get you up and running quickly.
