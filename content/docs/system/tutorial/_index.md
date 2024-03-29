---
title: donut.system tutorial
next: 01-your-first-system
weight: 1
---

[![Clojars Project](https://img.shields.io/clojars/v/party.donut/system.svg)](https://clojars.org/party.donut/system)

This tutorial guides you through the basics of using donut.system to structure
and interact with your application. By the end of it you'll be comfortable
creating and using systems, component groups, components, signal handlers, and
references. You'll understand exactly what's going on in this example:

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

The tutorial assumes that you know basic Clojure, and that you have an
understanding of [Clojure projects](https://www.braveclojure.com/appendix-a/)
and how to add libraries to your dependencies. [See the donut.system Clojars
page](https://clojars.org/party.donut/system) for specific dependency info.

You can view all source for tutorial examples [on
GitHub](https://github.com/donut-party/system/tree/main/dev/donut/examples/tutorial).
You can run these examples from a REPL.


OK enough jibber jabber! Let's go!

* [01: Your First System](01-your-first-system)
* [02: Implementing Behavior with Signal Handlers](02-implementing-behavior-with-signal-handlers)
* [03: Multiple Components and References](03-component-references)
* [04: Environment Configuraion](04-environment-configuration)
