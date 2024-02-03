---
title: "03: Multiple Components and References"
prev: 02-implementing-behavior-with-signal-handlers
---

{{< callout type="info" >}}

You can view all source for examples [on
GitHub](https://github.com/donut-party/system/tree/main/dev/donut/examples/tutorial)

{{< /callout >}}

Systems can contain multiple components, and those components can refer to each
other. In the example below we've updated our `APIPollerComponent` so that the
"data store" it uses (a humble atom) is placed in its own component.
`APIPollerComponent` uses `ds/ref` to refer to this new component so that it can
make use of it:

``` clojure {linenos=table,filename="dev/donut/examples/tutorial/03_multiple_components.clj"}
(ns donut.examples.tutorial.03-multiple-components
  (:require
   [donut.system :as ds]))

(def DataStoreComponent
  {::ds/start (fn [_] (atom nil))})

(def APIPollerComponent
  {::ds/start  (fn [{:keys [::ds/config]}]
                 (let [data-store (:data-store config)]
                   (future (loop [i 0]
                             (println "polling")
                             (reset! data-store i)
                             (Thread/sleep (:interval config))
                             (recur (inc i))))))
   ::ds/stop   (fn [{:keys [::ds/instance]}]
                 (println "stopping")
                 (future-cancel instance))
   ::ds/config {:interval   5000
                :data-store (ds/ref [:services :data-store])}})

(def system
  {::ds/defs
   {:services
    {:api-poller APIPollerComponent
     :data-store DataStoreComponent}}})
```

We've added `DataStoreComponent` to the system in the same way we added
`APIPollerComponent`, by placing it as a direct child of a component group
(`:services`) under `::ds/defs` so that donut.system can recognize it as a
component.

I want to make sure that the way things are named and organized in the system
map isn't confusing, so let's talk about that a bit.

## Component naming and organization

Let's look at the sytem map again:

``` clojure {linenos=table,linenostart=22,filename="dev/donut/examples/tutorial/03_multiple_components.clj"}
(def system
  {::ds/defs
   {:services
    {:api-poller APIPollerComponent
     :data-store DataStoreComponent}}})
```

`:services` is the name of a component group, and its value is a map with two
components. Their names are `:api-poller` and `:data-store`.

We haven't talked about component groups much, and that's because there's not
much to them. On a conceptual level, we use the term _component group_ to refer
a collection of components. On an implmentation level, a component group is just
a map. You can name component groups whatever you want. I'm actually struggling
to say more about them here because in a sense component groups aren't even
really a thing; your system definition is just maps of maps, and the term
component group is used to make it easier to keep track of what part of that
nested structure you're referring to.

Now that we have that awkward observation out of the way, let's discuss the way
things are named within a system. We refer to our components in different ways
depending on context:

* There's an API poller named `:api-poller`, `[:services :api-poller]`, and
  `APIPollerComponent`
* There's a data store named `:data-store`, `[:services :data-store]`, and
  `DataStoreComponent`

The key thing to keep in mind here is that, from the perspective of donut.system
functions, component definitions are entries within a `::ds/defs` map. When you
have a system that includes the component group `{:api-poller
APIPollerComponent}`, the var name `APIPollerComponent` is completely
irrelevant. All that donut.system sees is the map that `APIPollerComponent`
refers to; it isn't aware of the var name at all. Your var could be named
`MyWackyComponent🤪` and it would make no difference.

Whether or not you split your component definitions into vars is completely up
to you; it does not in any way change the way the library functions. Do whatever
makes it easy for you to understand and maintain your code.

By contrast, donut.system _does_ have a naming system for referring to
components within a system. For example, `ds/instance` will return a component
instance when given a _component id_:

```clojure
(def running-system (ds/start system))
(ds/instance running-system [:services :data-store])
```

Notice that we refer to the component instance using the vector `[group-name
component-name]`. We call this a _component id_ -- the component id is how you
refer to components within a system.

## References

`APIPollerComponent` contains a _reference_ (or just _ref_) on line 20:

``` clojure {linenos=table,linenostart=8,filename="dev/donut/examples/tutorial/03_multiple_components.clj",hl_lines=[13]}
(def APIPollerComponent
  {::ds/start  (fn [{:keys [::ds/config]}]
                 (let [data-store (:data-store config)]
                   (future (loop [i 0]
                             (println "polling")
                             (reset! data-store i)
                             (Thread/sleep (:interval config))
                             (recur (inc i))))))
   ::ds/stop   (fn [{:keys [::ds/instance]}]
                 (println "stopping")
                 (future-cancel instance))
   ::ds/config {:interval   5000
                :data-store (ds/ref [:services :data-store])}})
```

`(ds/ref [:services :data-store])` produces the vector `[:donut.system/ref
[:services :data-store]]`. References are our means of conveying component
instances into the signal handlers of another component. In this case, we want
`[:services :api-poller]` to have access to the instance produced by `[:services
:data-store]`.

To understand this, it helps to walk through what happens when you call
`ds/start` on this system:

1. The library calls the `::ds/start` signal handler for the `[:services
   :data-store]` component. The return value for the signal handler (an atom),
   becomes that component's instance and is stored in an updated system map
   under `[::ds/instances :services :data-store]`.
2. The library internally produces a _resolved component definition_ (or
   _resolved def_) for the `[:services :api-poller]` component. It takes the
   component and replaces every vector of the form `[::ds/ref component-id]`
   with the instance for `component-id`. In this case, the resolved def contains
   a `::ds/config` that includes the atom that was produced in step 1.
3. The library applies the `::ds/start` signal using the resolved def. Recall
   that signal handlers take one argument, a map, and that map includes the
   def's `::ds/config`. The `::ds/config` of the resovled def includes the data
   store atom, so that gets passed in to the `::ds/start` signal handler, and
   the handler body is able to update the state of that atom.

You might be wondering how the library knows to start `[:services :data-store]`
before `[:services :api-poller]` -- after all, if it started the components in
the reverse order then `[:services :api-poller]` would not be able to make use
of the `[:services :data-store]` instance. When you put references in your
system using `ds/ref`, the library internally constructs a directed graph of all
such references so that it can correctly apply signals in dependency order.

## Groups and local refs

We could rewrite `APIPollerComponent` to use a _local ref_, like this:

``` clojure {linenos=table,linenostart=8,filename="dev/donut/examples/tutorial/03_multiple_components.clj",hl_lines=[13]}
(def APIPollerComponent
  {::ds/start  (fn [{:keys [::ds/config]}]
                 (let [data-store (:data-store config)]
                   (future (loop [i 0]
                             (println "polling")
                             (reset! data-store i)
                             (Thread/sleep (:interval config))
                             (recur (inc i))))))
   ::ds/stop   (fn [{:keys [::ds/instance]}]
                 (println "stopping")
                 (future-cancel instance))
   ::ds/config {:interval   5000
                :data-store (ds/local-ref [:data-store])}})
```

Local refs refer to components within the same component group. Local refs can
be convenient in and of themselves, but their real value lies in how they open
up possibilities for creating reusable components and component groups. They're
similar to relative paths on a file system or relative URLs for web pages in
that way. Imagine being restricted to only ever being able to use absolute
paths; it would not be good!

So I guess I lied earlier when I said there's not much to component groups. In
fact, they let you do something very useful: use local refs.

## Summary

* Systems can contain multiple components, and those components can refer to
  each other
* System definitions are maps organized in a `::ds/defs` -> component group ->
  component hierarchy
* You use the `ds/ref` function to create refs. It takes one argument, a
  two-element vector of `[component-group-name component-name]`: a _component
  id_
* Signals are called in dependency order
* You can use local refs to refer to components that belong to the same group
