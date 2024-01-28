---
title: "03: Multiple Components and References"
prev: 02-implementing-behavior-with-signal-handlers
---



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
           :handler {::ds/start (fn [_]
                                  (fn [_req]
                                    {:status  200
                                     :headers {"ContentType" "text/html"}
                                     :body    "It's donut.system, baby!"}))}}}})
```

## Notes

* Add a second component
* Component naming / organization
* References
* The component graph, ordering, visualizing
* Component groups
