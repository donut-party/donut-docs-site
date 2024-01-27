---
title: Custom Signals
---

donut.system uses the terms "signal" and "signal handling" because component
behavior is extensible. Other component libraries use the term _lifecycle_,
which doesn't convey the sense of extensibility that's possible with
donut.system.

Out of the box, donut.system recognizes `::ds/start`, `::ds/stop`,
`::ds/suspend`, `::ds/resume`, and `::ds/status` signals, but it's possible to
handle arbitrary signals -- say, `:your.app/validate` or `:your.app/status`. To
do that, you just need to add a little configuration to your system:

``` clojure
(def system
  {::ds/defs    {;; components go here
                 }
   ::ds/signals {:your.app/status   {:order :topsort}
                 :your.app/validate {:order :reverse-topsort}}})
```

`::ds/signals` is a map where keys are signal names and values are configuration
maps. The configuration keys are:

**`:order`** values can be `:topsort` or `:reverse-topsort`. This specifies the
order that components' signal handlers should be called. `:topsort` means that
if Component A refers to Component B, then Component A's handler will be called
first; reverse is, well, the reverse.

**`:returns-instance?`** this determines whether the return value of the signal
handler should be used to update the system's instances, under `::ds/instances`.
You would set this to `false` if want to use the return value of the signal for
some purpose other than updating the instance. The `::ds/status` signal is set
up this way because its purpose is to return status information.

The map you specify under `::ds/signals` will get merged with your system's
default signal map, which is:

``` clojure
(def default-signals
  "which graph sort order to follow to apply signal, and where to put result"
  {::start   {:order             :reverse-topsort
              :returns-instance? true}
   ::stop    {:order             :topsort
              :returns-instance? true}
   ::suspend {:order             :topsort
              :returns-instance? true}
   ::resume  {:order             :reverse-topsort
              :returns-instance? true}
   ::status  {:order :reverse-topsort}})
```


