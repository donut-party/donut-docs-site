---
title: "03: Multiple Components and References"
prev: 02-implementing-behavior-with-signal-handlers
---

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
   ::ds/config {:interval 5000
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
instance:

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
   ::ds/config {:interval 5000
                :data-store (ds/ref [:services :data-store])}})
```

`(ds/ref [:services :data-store])` produces the vector `[:donut.system/ref
[:services :data-store]]`. References are our means of conveying component
instances into the signal handlers of another component. In this case, we want
`[:services :api-poller]` to have access to the instance produced by `[:produced
:data store]`.

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

We use references as placeholders for the
corresponding component instance. To understand this, let's walk through what
happens when you call `(ds/start system)`:



Our system 

Adding component definitions to a system is a matter of placing values in the
expected places in the system map. Defs are organized into component groups;
this system has three components:

* `[:env :http-port]`
* `[:http :server]`
* `[:http :handler]`

If you wanted to add more components to the system, you would do so be adding
more values to the `::ds/defs` map in the system map.

You may have noticed a couple things about the defs in the system above:

* The value `8080` at `[:env :http]` isn't a map! Nor is the function at `[:http
  :handler]`. No `::ds/start` signal handlers in sight. What gives?
* `ds/ref` and `ds/local-ref` are being used to encode the relationships among
  components

Let's look at each of these.

## Constant instance

So far, I've said that component definitions take the form of a map of signal
handlers. However, that's not the whole truth: you can use _any_ values to
define components. This is a valid system map:

```
(def constant-system
  {::ds/defs
   {:env {:env-name :production
          :http     {:port 8080}
          :db       {:username "bubba"
                     :password "crawdads"}}}})
```

This system has three components: `[:env :env-name]`, `[:env :http]`, and `[:env
:db]`. None of these definitions are maps with signal handlers.

In such cases, donut.system internally treats these component definitions as if
they're written like this:

```clojure
(def constant-system
  {::ds/defs
   {:env {:env-name {::ds/start (fn [_] :production)}
          :http     {::ds/start (fn [_] {:port 8080})}
          :db       {::ds/start (fn [_] {:username "bubba"
                                         :password "crawdads"})}}}})
```

That is, any time you provide a component definition that's not a map that
contains signal handlers, donut.system treats it as a "constant instance"


## References

## Component groups

## Notes

* Add a second component
* names: var names, component names, system positions
* References
* The component graph, ordering, visualizing
* Component groups
* constant instances
