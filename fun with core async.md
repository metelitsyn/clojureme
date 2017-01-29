# Fun with core.async

Let's play with clojure's core.async library.  

We start with requiting the 
```clojure
(ns asyncfun.core
  (:require [clojure.core.async :as async :refer[>! <! go chan close!]]

````
