---
title: "01: Your First System"
prev: _index
next: 02-implementing-behavior-with-signal-handlers
---

{{< callout type="info" >}}

You can view all source for examples [on
GitHub](https://github.com/donut-party/system/tree/main/dev/donut/examples/tutorial)

{{< /callout >}}

## Systems, components, and signal handlers

donut.system supports you in thinking about your application as a system of
interacting components, where:

* components are well-boundaried collections of processes and state that produce
  behavior to achieve some task.
* systems are collections of components

How does the library do this? By giving you a way to express the conceptual,
architectural organization of your application in code. Let's look at a very
simple example:

``` clojure {filename="dev/donut/examples/tutorial/01_your_first_system.clj"}
(ns donut.examples.tutorial.01-your-first-system
  (:require
   [donut.system :as ds]))

(def system
  {::ds/defs
   {:services
    {:printer {::ds/start (fn [_] (print "donuts are yummy!"))}}}})

(ds/signal system ::ds/start)
;; =>
donuts are yummy!
```

Going from the bottom up: we're calling `ds/signal` with two arguments, a
_system map_ and the _signal_ `::ds/start`. You can see that there's a map that
has the `:printer` with a map as its value, and that inner map includes the key
`::ds/start` -- the same keyword that we passed to `ds/signal`. Its function
gets called, and the result is that a true statement about donuts got printed.

How does this happen? What is the relationship between all these pieces -- the
`ds/signal` function, the structure of the `system` map, and the `::ds/signal`
keyword -- that produces the behavior we saw?

Conceptually, donut.system models system behavior in terms of sending and
responding to signals. (This is metaphorical: there's no network or sockets or
anything like that involved.) The system of components define _signal handlers_,
and when you "send" a signal to a system it _applies_ the signal by traversing
each component and calling its signal handler for that signal. This signal
handlers are what actually "do stuff" in your application.

This idea manifests in your code in the way components are defined. You define
components as maps that associate signal names (like `::ds/start`) with signal
handlers (functions like `(fn [_] (print "donuts are yummy!"))`). This is a
component definition:

``` clojure
{::ds/start (fn [_] (print "donuts are yummy!"))}
```

So when you call `(ds/signal system some-signal-name)`, it looks for all
components definition maps that have a key of `some-signal-name`, and calls the
corresponding function.

There's more to donut.system, but this is the foundation: giving you the tools
to structure your application as a system of components that produce behavior by
handling signals.

How might you modify this system definition so that your component can handle
other signals? Let's look at that next.

## How component definitions are organized



## What `ds/signal` returns

In the example above, the system has a component that defines a signal handler
for `::ds/start`

When you call `ds/signal`, it _applies_ the signal by traversing the system's
component definitions and calling the





In this chapter, you will learn:

* How donut.system represents systems and components as dat structures
* The structure of a system map and a component map
* why start?
  * ordering behavior
    * filling a cache
    * constructing a db thread pool

## Notes

* Start with data structure
* ::ds/defs
* what is a system?
* error messages if you get it wrong
* component which does nothing
* explain all the keys involved
* point out what's relevant, what's not
* donut.system exposes a lot, that's what makes it extensible

---

Next steps:

* Give another example
* Have an exercise
* Ask about expectations
