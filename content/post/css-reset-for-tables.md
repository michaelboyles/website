+++
author = "Michael Boyles"
title = "A CSS Grid Reset for Tables"
date = "2024-01-12"
description = "A guide for how to override the default CSS for <table> elements to use CSS grids instead"
tags = ["webdev"]
+++

Trying to style an HTML table is one of the few things that gives me flashbacks to how annoying web development used to
be back in the day. There are 8 different `display: table*` types which all have subtly inconsistent behaviour from
everything else. Want to add a border radius to a table row? Fuck you,
["the behavior on internal table elements is undefined"](https://developer.mozilla.org/en-US/docs/Web/CSS/border-radius#formal_definition).

<!--more-->

That was the straw that broke the camel's back for me today. I'm sick of dealing with issues like that, when working
with grids is comparatively such a pleasure. But I don't want to give up the semantic value of the `<table>` tag and its
children; I don't want to replace all my tables with `<div>`-based grids.

I've written a small CSS reset which uses CSS grids to replace most of the default `<table>` behaviour. I'll explain
each section individually. Jump to the bottom for the full code.

```css
table {
    display: grid;
    grid-template-columns: repeat(100, max-content);
}
```

The first line is self-explanatory, but I'm aware that the second one will have caused at least a few brows to crinkle.
If we never define how many columns a table has then everything will end up stacked in a single column. I intend to override
`grid-template-columns` for every table, but I wanted a default that wasn't going to be wrong almost all the time;
how often does a table genuinely only have one column?

The value 100 is fairly arbitrary. If you have fewer than 100 columns then any extra ones will all collapse to zero
width, so it doesn't really matter that it's significantly higher than most tables will have. I've seen some performance
issues with subgrids in the past, which is one reason I didn't use, say, 9999999.

If you add a `gap` then the extra columns will no longer collapse &mdash; you'll get a bunch of gaps between zero-width
columns &mdash; but if you're adding styles like `gap` then the idea is that you should also be overriding `grid-template-columns`.

```css
thead, tbody, tr {
    display: grid;
    grid-column: 1 / -1;
    grid-template-columns: subgrid;
}
```

Having `thead`, `tbody`, and `tr` span the entire table (`1 / -1`) and define their own subgrids achieves almost the
same thing as  if the `td` and `th` elements were direct children of the table. `display: contents` would achieve
something similar, but that would preclude being able to add a background colour or borders.

```css
td, th {
    display: block;
}
```

This just overrides the default `display: table-cell` in case that has inconsistent behaviour, which &mdash; let's
face it &mdash; it probably does.

```css
td[cellspan="2"] {
    grid-column: span 2;
}
```

It's very rare that I use cell spans, but you can achieve the same functionality using an attribute selector.
Unfortunately, you need one selector for every value of the span.

You could use similar selectors for row spans, but they won't work because a cell can't span across different subgrids.
You can fix that by changing the `thead, tbody, tr` selector to `display: contents`, but you'll lose the ability to
add a `background-color` and borders. You can work around that by adding those styles to the individual cells, but it's
kind of a faff.

I've literally never used `<colgroup>` so I'm going to pretend that it doesn't exist. I suspect there might be some
other wierd table features I'm forgetting.

The full code is below and I made [a CodePen to play with](https://codepen.io/lmayyqlf-the-bold/pen/YzgpNda?editors=1100)
as well.

```css
table {
  display: grid;
  grid-template-columns: repeat(100, max-content);
}
thead, tbody, tr {
    display: grid;
    grid-column: 1 / -1;
    grid-template-columns: subgrid;
}
td, th {
  display: block;
}
caption {
    grid-column: 1 / -1;
}
td[cellspan="2"] {
    grid-column: span 2;
}
td[cellspan="3"] {
    grid-column: span 3;
}
/* ... and so on */
```

Because this isn't going to work for every table, I might make this opt-in in future under a `.grid-table` class or
something. For now, I'm going to be applying this as a global style throughout a few projects to see what happens.
