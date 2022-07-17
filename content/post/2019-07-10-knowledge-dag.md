---
title: "Knowledge DAG"
date: "2019-07-10"
tags: ["k-dag", "ideas"]
---

If I have some specific problem to solve, I find technology that
seems relevant, then I scan the Internet for some
basic introduction, then some tutorial, then documentation.

My usual problem is that the issue I want to solve is usually not in the
introduction nor the tutorial. I rarely find ready-to-use solution
and unfortunately I tend to really dislike adjusting my idea to what you can easily
achieve with tutorial-like solutions. That means I usually have to
read large parts of the documentation just to find few little
details hidden somewhere in the middle.

Even though it's really the best way IMO, I believe that reading through
whole documentation of some project can be too time consuming. Our world is
accelerating and I believe that the ability to find a small subset of
information to reach specific goal may soon be a necessity.

Knowledge basics
----------------

Let's consider a basic example: learning hwo to calculate the **area of a triangle**.

First you may notice that there are few ways of calculating that area,
there are easier and harder ones. Some methods simplify calculations
based on some assumptions about triangles, others are totally universal.

Let's use the basic formula known from the school:

$$
A = \frac{1}{2} bh
$$

This formula may seem very simple - that's because we already know all the smaller
parts it consists of: we know how to multiply numbers, what a fraction is,
we also know how to measure the length in 2D space. We know it because we've learnt
it before. Even such basic things like the length of a line segment: you need to know
what is a point, a line, metric unit.

What we can see even from this small example is that knowledge terms form a pretty
well formed graph of dependencies:

![Dependency graph](https://g.gravizo.com/svg?
digraph G {
    ta[label="Triangle area"];
    mult[label="Multiplication"];
    ll[label="Line segment length"];
    ls[label="Line segment"];
    l[label="Line"];
    p[label="Point"];
    f[label="Fractions"];
    d[label="Division"];
    n[label="Real numbers"];
    c[label="Circle"];
    ll -> ta;
    mult -> ta;
    ls -> ll;
    l -> ls;
    p -> ls;
    f -> ta;
    d -> f;
    n -> d;
    n -> mult;
    n -> f;
    n -> ll;
    p -> c;
    ls -> c;
}
)

In this example graph it's pretty clear what you need to learn in order to know
how to calculate the area of a triangle. There's also an alternative term - **Circle**.
It does share some of it's dependencies with **Triangle area**.
And it's pretty clear that after mastering the **Circle** term,
you don't have to learn about **Points** and **Line segments** once again to master
the **Triangle area**.

Graph theory
------------

In more complex, real-world scenarios you'd see a huge cloud of terms and
dependencies between them all over the place. Even though it would be gigantic,
graphs have some really nice properties and we know efficient algorithms to
extract information from them.

For each node in such graph we could easily calculate it's complete dependency list
and order it so that you start from the basics,
go through more complex terms and finally get into the destination one.
It's also trivial to skip those paths which are irrelevant to the topic
being mastered.

Some terms may also have alternative learning paths. If you learn a new programming
language, it may be easier for you to start with a language you already know
rather than learning everything from scratch. A nice algorithm could find the best way
for you to master the new term, suggest alternative paths etc.

There are some nice opportunities in this area I'd like to explore. But that's a topic
for some future blog post.
