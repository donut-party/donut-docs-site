---
title: "04: Environment Configuration"
prev: 03-multiple-components-and-references
---

{{< callout type="info" >}}

You can view all source for examples [on
GitHub](https://github.com/donut-party/system/tree/main/dev/donut/examples/tutorial)

{{< /callout >}}

Most applications need to read configuration values from their environment. Such
configuration typically includes connection parameters for databases and API
services. What's more, configuration values vary per environment (local, qa,
prod, etc), so you need some way of specifying what environment your system is
running in. Below is an example of how to handle this with donut.system:

``` clojure {linenos=table,filename="dev/donut/examples/tutorial/04_environment_configuration.clj"}
(ns donut.examples.tutorial.04-environment-configuration
  (:require
   [aero.core :as aero]
   [clojure.java.io :as io]
   [donut.system :as ds]))

(defn env-config [& [profile]]
  (aero/read-config (io/resource "config/env.edn")
                    (when profile {:profile profile})))

(def DataStoreComponent
  {::ds/start (fn [_] (atom nil))})

(def APIPollerComponent
  {::ds/start  (fn [{:keys [::ds/config]}]
                 (let [{:keys [data-store source]} config]
                   (future (loop [i 0]
                             (println (str "polling " source))
                             (reset! data-store i)
                             (Thread/sleep (:interval config))
                             (recur (inc i))))))
   ::ds/stop   (fn [{:keys [::ds/instance]}]
                 (future-cancel instance))
   ::ds/config {:interval   (ds/ref [:env :api-poller :interval])
                :source     (ds/ref [:env :api-poller :source])
                :data-store (ds/ref [:services :data-store])}})

(def base-system
  {::ds/defs
   {:env      {}
    :services {:api-poller APIPollerComponent
               :data-store DataStoreComponent}}})

(defmethod ds/named-system :base
  [_]
  base-system)

(defmethod ds/named-system :dev
  [_]
  (ds/system :base {[:env] (env-config :dev)}))


(defmethod ds/named-system :prod
  [_]
  (ds/system :base {[:env] (env-config :prod)}))
```

The strategy here is:

1. Use another library, [aero](https://github.com/juxt/aero), to transform
   config files into Clojure data structures for your application. aero is EDN,
   but with some enhancements, including a little syntax sugar for incorporating
   environment variables and for producing different values based on the
   `:profile` you pass in. `env-config` uses aero on line 8; see the aero docs
   for more info.
2. Create _named systems_ that introduce per-environment configuration by
   modifying a base system. The `:dev` and `:prod` named systems use the
   `env-config` function to read environment-specific values and place them in
   the `:env` component group.
2. Create components that pull their environment configuration from the `:env`
   component group. `APIPollerComponent` has refs for `[:env :api-poller
   :interval]` and `[:env :api-poller :source]` on lines 24 and 25. These are
   _deep refs_ and I cover them below.
  
To fully understand this strategy, we need to understand:
  
* Constant instances
* Deep refs
* Named systems

## Constant instances

If a component definition is anything other than a map that includes signal
handler keys, then it's treated as a "constant instance." Observe:

``` clojure
(ns donut.examples.tutorial.04-constant-instance
  (:require
   [donut.system :as ds]))

(def system
  {::ds/defs
   {:env      {:db-conn-string "//localhost:5032etcetc"}
    :services {:db {::ds/start  (fn [{:keys [::ds/config]}]
                                  (prn "db-conn-string" (:db-conn-string config)))
                    ::ds/config {:db-conn-string (ds/ref [:env :db-conn-string])}}}}})

(ds/start system)
```

If you evaluate all of this at your REPL, you'll see the following get printed:

```
"db-conn-string" "//localhost:5032etcetc"
```

What's happening here is the `[:services :db]` component is referencing `[:env
:db-conn-string]`. Recall that references are our means of conveying component
instances into the signal handlers of another component.

That means that the component instance for `[:env :db-conn-string]` is the
string `"//localhost:5032etcetc"`. However, the component definition for `[:env
:db-conn-string]` is not a map with signal handlers like we've seen so far; it's
the string `"//localhost:5032etcetc"`. 

This violates our understand of how component definitions work. So far, learned
that component definitions are maps of signal handlers, and that signal handler
return values become component instances. You would expect that for this to
work, you would have to use the following component definition for `[:env
:db-conn-string]`:

```
{::ds/start (constantly "//localhost:5032etcetc")}
```

However, because it's such a common use case to want to include such constant
instances, donut.system was designed to support you in including the value
directly, rather than having to wrap it in a signal handler.

In practice, this means that components and component groups can essentially be
paths in your system that house configuration. The full system at the top of
this page takes this approach: it uses aero to read a config file, generating a
Clojure map in the process. That Clojure map then gets placed under the `:env`
component group. Other components can then access that configuration via refs.

## Deep refs

`APIPollerComponent` at the top of the page includes these two refs:

``` clojure
(ds/ref [:env :api-poller :interval])
(ds/ref [:env :api-poller :source])
```

So far, we've only seen refs that take two-element vectors of the form
`[component-group-name component-name]`. But in the code, the vectors have
_three_ elements.

Refs can actually take any number of elements; you can think of the vector you
pass in as being used to perform a `get-in` on your system's instances:

```
(get-in (::ds/instances system) [:env :api-poller :interval])
```

If a component instance is a deeply-nested map, you can use refs to refer to any
path within that map. This also works with vectors, as you can use `get-in` on
vectors:

``` clojure
(get-in [[:a :b] [:c :d]] [1 0])
;; =>
:c
```

## Named systems

The multimethod `ds/named-system` serves as a system definition registry. We can
then use the function `ds/system` to retrieve a system definition and optionally
override component definitions. We see this in the example at the top of the
page:

``` clojure{linenos=table,linenostart=28,filename="dev/donut/examples/tutorial/04_environment_configuration.clj"}
(def base-system
  {::ds/defs
   {:env      {}
    :services {:api-poller APIPollerComponent
               :data-store DataStoreComponent}}})

(defmethod ds/named-system :base
  [_]
  base-system)

(defmethod ds/named-system :dev
  [_]
  (ds/system :base {[:env] (env-config :dev)}))


(defmethod ds/named-system :prod
  [_]
  (ds/system :base {[:env] (env-config :prod)}))
```

We register a system definition named `:base`, then we build on that system in
the `:dev` and `:prod` named systems, overriding the `:env` component groups
with environment-specific configurations produced by the `env-config` function.

Together, all of these pieces allow you to define the main structure of your
system -- the components that produce the behavior your care about -- while
giving you the flexibility to configure these components for different
environments.

Where do you actually put all this? My recommendation is to create a
`your-project.system` namespace and put your system definitions there.

And with that, you now have all the basics you need for effectively using
donut.system in a real project! Woo!

## Summary

* Any component definition that isn't a map of signal handlers is treated as a
  _constant instance_
* You can use `ds/named-system` to register different system definitions
* Refs can actually refer to any part of a system's `::ds/instances` that's
  reachable via `get-in`
