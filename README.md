# re-thread

[![Clojars Project](https://img.shields.io/clojars/v/com.yetanalytics/re-thread.svg)](https://clojars.org/com.yetanalytics/re-thread)

A tiny Clojurescript library for interacting with re-frame when it's running in a Web Worker.

## Why

re-frame is great for heavy lifting in the browser, but sharing a single thread with the UI can still be problematic. By shoving re-frame's state and events (and any other event/comms stuff like websockets) into a worker thread, we can keep the main browser thread nice and responsive.

## How

re-thread abstracts re-frame's subscription process over the Web Workers API messaging system. View components in the main thread "subscribe" to re-frame subscriptions defined in the worker, where reagent's `track!` function is used to watch them and transmit updates back. Disposals of subscriptions in the main thread are monitored, resulting in disposal of the corresponding tracks in the worker thread.

## Usage

In your Web Worker re-frame app:

``` clojure
(ns your-worker-app.core
  (:require
   [re-thread.worker :refer [listen!]]
   ...))

(listen!)

```

Your client/host application will consist of only views and handler calls, delegating everything else to the worker.

To initialize the worker, use `re-thread.client/init`:

``` clojure
(ns your-client-app.core
  (:require
   [re-thread.client :refer [init]]
   ...))


;; Remember, in reloadable environments, you only want to run init once.
(defonce init-worker
  (delay
   ...
   (init "/js/compiled_worker.js" render) ;; optional callback
   ))

@init-worker

```

Then, in your components, you can pretty much forget that things are happening in a worker, and use `re-thread.client/dispatch` and `re-thread.client/subscribe` in the manner to which you are accustomed:

``` clojure
(ns your-client-app.views.counter
  (:require [re-thread.client :refer [dispatch subscribe]]))

(defn counter []
  (let [v (subscribe [:counter/value] 0)] ;; <- note that you can provide a default value..
    (fn []
      [:div
       [:p "Counter Value: " (str @v)
        [:span
         [:button
          {:on-click #(dispatch [:counter/inc-value!])} ;; <- registered in the worker
          "Increment"]
         [:button
          {:on-click #(dispatch [:counter/reset-value!])}
          "Reset"]]]])))

```

## Example App

I set up the re-frame todomvc example in a webworker, try it out in the `examples` directory!

## Gotchas

* Since all of the app state is contained in the worker process, you should take care not to write view code that fails if a subscription returns nil. Using the optional second arg of `subscribe` can help with that.
* Though you get all of re-frame's subscription caching + other goodness in the worker, remember that each novel subscription result has to be serialized and sent back from the worker (once per sub id, client subscriptions are also deduplicated). Large results may hurt performance.
* Doesn't seem to work without advanced compilation on Safari 10, fine in Chrome/Firefox. The initial subscription messages to the worker are lost with no error. Minified builds work fine though.

## TODO:

* Docs
* Batching/caching worker -> client subscription results
* Multiple Workers
* Extensible Messaging


## License

Copyright © 2017 Yet Analytics Inc.

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
