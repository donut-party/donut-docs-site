---
title: "03: Multiple Components and References"
prev: 02-implementing-behavior-with-signal-handlers
---

Systems can contain multiple components, and those components can refer to each
other. Below is a stripped down example of how you might organize a web
application into components. Later we'll look at incorporating
`APIPollerComponent` from previous sections.

``` clojure {linenos=table,filename="dev/donut/examples/tutorial/03_multiple_components.clj"}
(ns donut.examples.tutorial.03-multiple-components
  (:require
   [donut.system :as ds]
   [ring.adapter.jetty :as rj]))

(def system
  {::ds/defs
   {:env  {:http-port 8080}
    :http {:server  {::ds/start  (fn [{:keys [::ds/config]}]
                                   (let [{:keys [handler options]} config]
                                     (rj/run-jetty handler options)))
                     ::ds/stop   (fn [{:keys [::ds/instance]}]
                                   (.stop instance))
                     ::ds/config {:handler (ds/local-ref [:handler])
                                  :options {:port  (ds/ref [:env :http-port])
                                            :join? false}}}
           :handler (fn [_req]
                      {:status  200
                       :headers {"ContentType" "text/html"}
                       :body    "It's donut.system, baby!"})}}})
```

This system has three components:

* `[:env :http-port]`
* `[:http :server]`
* `[:http :handler]`

The `[:http :server]` component depends on both `[:env :http-port]` and `[:http
:handler]`:

``` mermaid
graph TB
server(":http :server") --> handler(":http :handler")
server(":http :server") --> env-port(":env :http-port")
```

This system allows us to configure the server's request handler and options like
what port to run on.

You may have noticed a couple things about the system:

* The value `8080` at `[:env :http]` isn't a map! Nor is the function at `[:http
  :handler]`. What gives?
* `ds/ref` and `ds/local-ref` are being used to encode the relationships among
  components

Before we look at these two topics, try starting your system with `(ds/start
system)`. It should start a little web server that serves up the text `"It's
donut.system, baby!"` when you visit
[http://localhost:8080](http://localhost:8080).

## Constant instances



## References

## Component groups

## Notes

* Add a second component
* constant instances
* References
* The component graph, ordering, visualizing
* Component groups
