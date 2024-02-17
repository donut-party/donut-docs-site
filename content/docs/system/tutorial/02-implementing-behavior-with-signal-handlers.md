---
title: "02: Implementing Behavior with Signal Handlers"
prev: 01-your-first-system
---

{{< callout type="info" >}}

You can view all source for examples [on
GitHub](https://github.com/donut-party/system/tree/main/dev/donut/examples/tutorial)

{{< /callout >}}

Components define signal handlers that respond to signals to produce behavior,
where "behavior" is a combination of processes and state that yields some
desired state.

Your components can implement arbitrary signal handlers, but generally they'll
implement `::ds/start` and, if needed, `::ds/stop`. The example below has a
component that simulates polling an API: the `::ds/start` handler creates a
"poller" (just a future) that "polls" every 5 seconds in a loop, and the
`::ds/stop` handler stops the poller:

``` clojure {linenos=table,filename="dev/donut/examples/tutorial/02_signal_handlers_a.clj"}
(ns donut.examples.tutorial.02-signal-handlers-a
  (:require
   [donut.system :as ds]))

(def APIPollerComponent
  {::ds/start (fn [_]
                (let [poll-data (atom nil)]
                  {:poller    (future (loop [i 0]
                                        (println "polling")
                                        (reset! poll-data i)
                                        (Thread/sleep 5000)
                                        (recur (inc i))))
                   :poll-data poll-data}))
   ::ds/stop  (fn [{:keys [::ds/instance]}]
                (println "stopping")
                (update instance :poller future-cancel))})

(def system
  {::ds/defs
   {:services
    {:api-poller APIPollerComponent}}})

(comment
  ;; evaluate these manually at your REPL
  (def running-system (ds/signal system ::ds/start))
  (ds/signal running-system ::ds/stop)

  ;; alternatively, use these convenience functions:
  (def running-system (ds/start system))
  (ds/stop running-system)


  (::ds/instances running-system))
```

There are many things going on here that we need to examine:

* The component's behavior, using the core Clojure functions `future` and `future-cancel`
* The value returned by signal handlers
* The way we hold on to the running poller so that we can then stop it

## Component behavior

It will help to understand what the component is actually *doing*.
`APIPollerComponent` has a `::ds/start` signal handler, a function with this
body:

``` clojure
(let [poll-data (atom nil)]
  {:poller    (future (loop [i 0]
                        (println "polling")
                        (reset! poll-data i)
                        (Thread/sleep 5000)
                        (recur (inc i))))
   :poll-data poll-data})
```

This does three things:

1. Creates an atom, `poll-data`, to store the "results" of an API call
2. Creates a `future` that has a loop that calls `reset!` on the atom (and
   prints a message) every 5 seconds
3. Returns a map with the keys `:poller` and `:poll-data`

Here's part of the docstring for `future`:

> Takes a body of expressions and yields a future object that will
> invoke the body in another thread

The key idea here is that we want our poller to execute in the background on a
separate thread. In a real system, you might have many such background processes
that are retrieving data or performing computations. In this example, using
`future` is a convenient way to place the work on a separate thread. If you'd
like to learn more about futures and concurrent programming, check out [The
Sacred Art of Concurrent and Parallel
Programming](https://www.braveclojure.com/concurrency/).

`future` returns a _future object_, and if you call `future-cancel` on it then
you will stop the future's body from executing. This `future` / `future-cancel`
combo is an easy, lightweight way to create a background process in your system,
and using an atom like `poll-data` is one way to convey your component's state
to other parts of the system. (In a later chapter you'll see examples of how
components can refer to each other so that one component can make use of the
data/objects/results provided by another component.)

Both the future we create and the `poll-data` atom are returned by the
`::ds/start` signal handler in the map `{:poller ..., :poll-data poll-data}`.
Let's look at how that get's used.

## Signal handlers return component instances

When you call `(ds/signal system signal-name)`, the `ds/signal` function
traverses all component definitions in `system` and calls the signal handler
corresponding to `signal-name` for that component. The signal handler's return
value is the component's _instance_ and it gets associated back into the system
map under the `::ds/instances` key. After `ds/signal` is done calling all signal
handlers, it returns an updated system map that includes these component
instances:

``` clojure {filename=REPL}
;; `system` is the original system definition map, and has no instances
(::ds/instances system)
;; =>
nil

;; `running-system` was produced by `(ds/signal system ::ds/start)` and 
;; includes instances
(::ds/instances running-system)
;; =>
{:services
 {:api-poller
  {:poller #<Future@4243c35a: :pending>, :poll-data #<Atom@124804ba: 1>}}}
```

Notice that the location of a component's instance in the system map corresponds
to the location of a component's definition. The component definition
`APIPollerComponent` lives in the system map under `[::ds/defs :services
:api-poller]`, and its instance lives in the system map under `[::ds/instances
:services :api-poller]`.

You can make use of this fact to retrieve or interact with a component's
instance: if the component's definition is located in the system map under
`[::ds/defs :web :worker-1]`, then its instance will be located under
`[::ds/instances :web :worker-1]`.

## Component organizational structure

This organizational structure is a key aspect of donut.system's design: a system
stores different facets of a component (its definition, its instance, and more)
in the same relative location across different maps.

Internally, donut.system's implementation relies on this structure to manage the
way it conveys a component's instance to its signal handlers. Externally, this
organization structure is meant to make it easier for developers like yourself
to find information about a component.

For example, there are many reasons why you might want to interact with a
component's instance: you might want to inspect an instance in a running system
to debug an issue, you might want to write assertions about the value of an
instance in a test, etc. donut.system's organizational structure (hopefully)
makes it obvious how to retrieve the instance you're interested in.

If you need to get an instance, the library also includes the `ds/instance`
function:

``` clojure
(ds/instance running-system [:services :api-poller])
```

In addition to being a bit clearer, using the `ds/instance` function has the
advantage that it will throw an exception if the instance you're looking for
isn't defined. This can help prevent typos.

## Accessing instances in signal handlers

Signal handlers are passed one argument, a map. This map includes the key
`::ds/instance`, whose value is the component's instance. This is how signal
handlers can access their component's instance, and you can see it in the
`APIPollerComponent` example:

``` clojure {linenos=table,linenostart=5,filename="dev/donut/examples/tutorial/02_signal_handlers_a.clj"}
(def APIPollerComponent
  {::ds/start (fn [_]
                (let [poll-data (atom nil)]
                  {:poller    (future (loop [i 0]
                                        (println "polling")
                                        (reset! poll-data i)
                                        (Thread/sleep 5000)
                                        (recur (inc i))))
                   :poll-data poll-data}))
   ::ds/stop  (fn [{:keys [::ds/instance]}]
                (println "stopping")
                (update instance :poller future-cancel))})
```

The `::ds/stop` signal handler at line 14 destructures `::ds/instance` from its
argument. The instance is this map:

``` clojure
{:poller    the-future-running-the-poller 
 :poll-data the-poll-data-atom}
```

The signal handler returns `(update instance :poller future-cancel)` -- this
returns the instance map, updated so that the poller future is cancelled.

We could have written the signal handler like this:

``` clojure
(fn [{:keys [::ds/instance]}]
  (println "stopping")
  (future-cancel (:poller instance)))
```

If we did that, then `future-cancel`'s return value of `true` would be used as
the instance for `[:services :api-poller]`. That might be fine, but generally
it's more useful to retain as much instance data as possible when applying
signal handlers. This way, for example, you can examine the value of `poll-data`
after the system has stopped.

## Component configuration

What if you wanted to make your polling interval configurable? Here's how you
might do that:

``` clojure {linenos=table,linenostart=5,filename="dev/donut/examples/tutorial/02_signal_handlers_b.clj"}
(def APIPollerComponent
  {::ds/start  (fn [{:keys [::ds/config]}]
                 (let [poll-data (atom nil)]
                   {:poller    (future (loop [i 0]
                                         (println "polling")
                                         (reset! poll-data i)
                                         (Thread/sleep (:interval config))
                                         (recur (inc i))))
                    :poll-data poll-data}))
   ::ds/stop   (fn [{:keys [::ds/instance]}]
                 (println "stopping")
                 (update instance :poller future-cancel))
   ::ds/config {:interval 5000}})
```

You can include a `::ds/config` key in your component definition, as we do on
line 17. This gets passed in to signal handlers; the `::ds/start` handler
destructures `::ds/config` from its argument on line 6. Now the sleep interval
is no longer hard-coded.

But isn't it still kind of hard-coded? What if you want to change it?

### Modifying configs with core functions

There are a few different ways that you could do this, and they're all derived
from the fact that your system definition is just a big ol' nested Clojure map.
Here's one way you could do it:

``` clojure
(def running-system
  (ds/signal (assoc-in system [::ds/defs :services :api-poll ::ds/config :interval] 1000)
             ::ds/start))
```

Here we've used `assoc-in` to produce an updated system map with a new value for
the poller's `:interval`. Because the interval value is now accessible within
the system map (as opposed to living inaccessibly within the body of the
`future`), it can be modified using everyday Clojure functions.

### Modifying configs with overrides

Because this is such a common need, there's a more succinct syntax you can use
to _override_ these values:

``` clojure
(def running-system
  (ds/start system
            {[:services :api-poll ::ds/config :interval] 1000}))
```

`ds/start` is a helper that sends the `::ds/start` signal to a system, and
allows you to optionally provide a map of _overrides_. Each key in the map is a
path to a location under `::ds/defs` in your system map, and the value is the
new value you want at that position. It's like you're doing this:

```clojure
(assoc-in system [::ds/defs :services :api-poll ::ds/config :interval] 1000)
```

except for every key/value pair found in the overrides map.

You might be wondering if you can override signal handlers, and the answer is
yes, you can:

``` clojure
(def running-system
  (ds/start system
            {[:services :api-poll ::ds/start] (fn new-signal-handler [_] ...)}))
```

This is useful for mocking out components for testing, which we'll look at more
later.

The general capability here is that you can use the overrides map to
conveniently express that you want to perform any number of `assoc-in` calls on
`::ds/defs` in the system map. One of the main use cases for this is updating
component configs, but the underlying mechanism purely operates on maps and is
blind to the semantics of what you're trying to achieve.

### `ds/system` takes overrides, too

You can also produce a new system definition with overrides using the
`ds/system` function:

``` clojure
(def test-system
  (ds/system system
             {[:services :api-poll ::ds/start] (fn new-signal-handler [_] ...)}))
```

This doesn't send a signal to the system, it just produces a system map.

That covers it for signal handling! This should be enough to allow you to get
you writing components that can start and stop themselves, and who doesn't want
that?? Next, let's look at how to work with more than one component.

## Summary

* Signal handlers return _component instances_
* A component's instance is passed in to its signal handler via the
  `::ds/instance` key
* You typically use `::ds/start` and `::ds/stop` signal handlers to start and
  stop a component
* You can pass in component configuration with the `::ds/config` key
* You can modify component definitions with regular ol' core functions or with
  override syntax

## Further reading

* [Custom Signals]({{< ref "/docs/system/guides/custom-signals" >}})
