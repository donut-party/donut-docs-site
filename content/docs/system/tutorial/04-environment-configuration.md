---
title: "04: Environment Configuration"
prev: 03-multiple-components-and-references
---

Most applications need to read configuration values from their environment. Such
configuration typically includes connection parameters for databases and API
services. What's more, configuration values vary per environment (local, qa,
prod, etc), so you need some way of specifying what environment your system is
running in. Below is an example of how to handle this with donut.system:

``` clojure
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

The overall strategy being employed here is:

1. Use an external library, [aero](https://github.com/juxt/aero), to transorm
   config files into Clojure data structures for your application. aero is EDN,
   but with some enhancements, including a little syntax sugar for incorporating
   environment variables. `env-config` uses aero on line 8.
2. Create _named systems_ that introduce per-environment configuration by
   modifying a base system. The `:dev` and `:prod` named systems use the
   `env-config` function to read environment-specific values and place them in
   the `:env` component group.
2. Create components that pull their environment configuration from the `:env`
   component group. `APIPollerComponent` has refs for `[:env :api-poller
   :interval]` and `[:env :api-poller :source]` on lines 24 and 25. These are
   _deep refs_ and I cover them below.
   
This may look like a lot, but don't worry: it builds on everything we've done so
far and only introduces a few new concepts, which we are going to examine in detail:

* Constant instances
* Deep refs
* Named systems



## Notes

You need to configure based on your environment: database connection info, API
servers, etc.

* Constant instances
* Deep refs
* Named systems
* How to include environment variables and other configuration 
* How to create different configurations for dev, prod, etc


* organizing with named-system
* constant instances and deep refs
* system namespace
* configuration: all the different methods
  * runtime
  * aero
