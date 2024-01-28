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
  behavior
* systems are collections of components

How does the library do this? By giving you a way to express the conceptual,
architectural organization of your application in code. Let's look at a very
simple example:

``` clojure {linenos=table,filename="dev/donut/examples/tutorial/01_your_first_system.clj"}
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

Going from the bottom up: on line 10 we're calling `ds/signal` with two
arguments, a _system map_ and the _signal_ `::ds/start`. On line 8 you can see
that there's a map that has the key `:printer` with a map as its value, and that
inner map includes the key `::ds/start` -- the same keyword that we passed to
`ds/signal`. Its function gets called, and the result is that a true statement
about donuts got printed.

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

When I say "it looks for all component definition maps", I mean that component
definition maps must be organized in a particular way within the system map for
them to be found and used. The general structure is:

``` clojure
{::ds/defs
 {:group-1
  {:component-1 component-def-map
   :component-2 component-def-map}
  :group-2
  {:component-3 component-def-map}}}
```

The value of `::ds/defs` is map whose keys are group names, and and whose values
are component groups. A component group is a map whose keys are component name,
and whose values are component maps.

For a map to be treated as a component definition, it must be located at the
correct place within these nested maps. It must be accessible via get-into using
a 3-element path like `[::ds/defs :group-1 :component-1]`; this won't work:

``` clojure
{::ds/defs
 {:group-1
  {:sub-group-1 component-def-map}}}
```

There's more to donut.system, but this is the foundation: it gives you the tools
to structure your application as a system of components that produce behavior by
handling signals.

How might you modify this system definition so that your component can handle
other signals? Let's look at that next.


## Notes

* real-world example
  * system namespace
* testing
* donut.system exposes a lot, that's what makes it extensible
* configuration: all the different methods
  * runtime
  * aero
* dev tools: inspecting, maintaining
* REPL development
* named-system multimethod
  **
* doesn't have to be stateful
* error messages if you get it wrong

---

Next steps:

* Give another example
* Have an exercise
* Ask about expectations
