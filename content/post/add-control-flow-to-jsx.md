+++
author = "Michael Boyles"
title = "Adding control flow to JSX with TypeScript"
date = "2021-06-13"
description = "In this post, I explain how if-else statements can be added to JSX in a React project"
tags = ["project"]
location = "Chester, England"
+++

In any non-trivial React project, one of the things you'll eventually need to do is render something conditionally.
You [can't use if-else statements within JSX](https://react-cn.github.io/react/tips/if-else-in-JSX.html). You can use
the conditional (“ternary”) operator but I often find that it produces code that's hard to read. I've developed a
transform which adds special meaning to certain JSX tags, allowing you to write `<If>` and `<Else>` blocks which behave
intuitively.

<!--more-->

[JSX](https://facebook.github.io/jsx/) is a language extension for JavaScript developed by Facebook. In practice, it's
used as an almost intrinsic [part of React](https://reactjs.org/docs/jsx-in-depth.html), though there's no tight
coupling between the two: you could use JSX without React and vice versa.

JSX is basically syntactic sugar which allows us to write more expressive code that's closer to the HTML we're looking
to generate. Take these two examples:

```jsx
const div = <div id="msg">Hello World!</div>;
const customComponent = <MyClock type="analogue" />;
```

These could be converted to:

```javascript
const div = React.createElement("div", {id:"msg"}, "Hello World!");
const customComponent = React.createElement(MyClock, {type:"analogue"});
```

You could also use a different transpiler to produce some other output unrelated to React. The JSX specification states
that JSX comes “without any defined semantics”. In simple terms, what that means is that a transpiler can decide what
should be generated. It doesn't need to be invocations of React functions and it doesn't even *necessarily* need to be
anything to do with generating HTML.

JSX mostly succeeds at its goal of making our code more expressive but, because the authors of the JSX spec decided -
in my opinion, correctly - that it should be decoupled from React, they surrendered the ability to include any features
which would specifically benefit React. In other words, by not defining *any* semantics, they couldn't define any
*React-specific* semantics.

Take this example which uses a conditional operator to decide which `<div>`s should be rendered[^1].

```jsx
<div id="wrapper"> 
    { /* Not so bad: */ }
    { someCondition ? <div id="myDiv">Div text</div> : null }
    { /* Worse: */ }
    {
        (someValue === 4 && myStr.includes("search str") ? 
            <div id="anotherDiv"
                 className="myClass"
                 draggable
            >
                The condition was true
            </div> : 
            <div>The condition was false</div> 
    }
</div>
```

Maybe not everyone will agree but I find it very easy to miss the question mark and colon tokens with everything else
that's going on. The conditional operator wasn't really designed to be used this way, so it's hardly surprising that it
doesn't hold up very well. The reason we're limited to using the conditional operator
(or [the workarounds described here](https://react-cn.github.io/react/tips/if-else-in-JSX.html)) is because of how JSX's
grammar is defined: in JavaScript `if` is a statement and JSX deals with expressions[^2]. 

I've seen a few attempts in various projects to try to handle this more gracefully. They usually end up looking
something like this, where `<If>` is a custom component in which the render method evaluates the condition before
returning either the children or null.

```jsx
<If condition={Boolean(someObject)}>
    <MyComponent foo={getFoo()} bar={someObject.bar} />
</If>
```

I find this much more expressive than a conditional operator. The implementation isn't difficult to write either, but it
has one notable problem which is better understood by looking at the generated code:

```javascript
React.createElement(If, { condition: Boolean(someObject) },
    React.createElement(MyComponent, {foo: getFoo(), bar: someObject.bar } })
)
```

Since all of the arguments to the first `createElement` invocation must be evaluated before entering the render method
of `<If>`, both `getFoo()` and `someObject.bar` are evaluated as well. Calling `getFoo()` is unlikely to be too much of
a problem unless it's expensive but it's still wasted effort. Accessing `someObject.bar` is worse, though. Since
`someObject` may be falsy (i.e. undefined or null), trying to access `bar` will throw an error. The conditional operator
short-circuits but our special component doesn't.

How can we solve that? It's as simple as changing our transformation. We need to make our transpiler aware of these
additional semantics. We need to teach it to understand `<If>` blocks.

I presume that most tools used for transpiling JavaScript (e.g. [Babel](https://babeljs.io/)) are extensible enough to
achieve this but I chose TypeScript. The TypeScript compiler
[can parse JSX natively](https://www.typescriptlang.org/docs/handbook/jsx.html)
and [with a little prodding](https://github.com/cevek/ttypescript) exposes a neat way of manipulating the AST.

Simply put, we can replace our usages of `<If>` at compile-time to use a conditional operator instead, as if by magic.
After transformation, the previous example compiles to the following code. The difference is subtle but important.

```javascript
Boolean(someObject) ?
    React.createElement(MyComponent, {foo: getFoo(), bar: someObject.bar } })
    : null
```

I also added support for `<Else>` conditions which I don't think would be possible at all without a transformation.
The `<Else>` block must immediately follow the closing `</If>`, excluding whitespace[^3].

```jsx
<If condition={someCondition}>
    <div>The condition is true</div>
</If>
<Else>
    <div>The condition is false</div>
</Else>
```


By using transformation, we can achieve the expressiveness we want but without losing the desirable properties of
short-circuiting.

The [full code is available on GitHub](https://github.com/michaelboyles/if-jsx). The transformation itself is only
~150 lines.

[^1]:  The three attributes arranged vertically are done so for the sake of the example. In this specific case, it might
be cleaner to place them all on one line. In reality, seeing 10 or more attributes is not uncommon so this vertical
style is used a lot.

[^2]: In other languages [like Kotlin](https://kotlinlang.org/docs/control-flow.html), `if` can act as an expression.
This effectively removes the need for a conditional operator

[^3]: This unfortunately means you can't add a comment between the two blocks such as `{/*Otherwise, do this*/}`. I
could whitelist that specific case but it doesn't seem important and I'd rather keep the behaviour simple.
