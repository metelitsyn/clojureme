# Fun with core.async

Let's play with clojure's core.async library.  

We start by adding a depndency in the project.clj
```clojure
[org.clojure/core.async "0.2.385"]
```
and requiring the frequently used function names 
```clojure
(ns asyncfun.core
  (:require [clojure.core.async :as async :refer[>! <! go chan close!]]
```
We want to explore a little bit how core.async works and how we can use it to deal with random events. 'Random' will mean in this context that neighter do we know the length of input

Let's write a simple emitter function.

The emitter will generate new messages of the form {:tag <some number> :msg <some string>} at random time intervals between 0 and 3 seconds.
n
We want to realize a following application logic. A data source emits messages sequentially at random intervals. A good example for such a data source is a network socket or a function which polls a web service. We assume that each arriving chunk of data can be processed independently from others. However the processing time may be longer than intervals between data chunks. We want the processing to start as soon as data chunk arrieved.  
First let's write a data emitter.
```clojure
(defn random-data-source [N]
  (let [out (chan)]
    (go 
      (dotimes [ n N]
        (do
          (<! (async/timeout (rand-int 3000)))
          (>! out {:id n}))))
    out))
```
It is a function which returns a channel and attempts to pass a hashmap of the form {:id <number>} down the channel at random time. The input argument N limits the number of produced messages. Note that a call to this function will return immideately and does not block the caller. We can use a simple echo function to monitor a channel.
```clojure
(defn echo [in]
    (go (while true (prn (str "#echo:" (<! in)))))))
```
It takes a chanel to be monitored as an input and listens for messages on this channel forever.
We can now chain the emitter and monitor  
```clojure
(-> 3 random-data-source echo)
```
This will produce
```
"#echo: {:id 0}"
"#echo: {:id 1}"
"#echo: {:id 2}"
```
Usually we want to process incomming data. Assume we have a function which takes much longer than the intervals between the arriving of new data. For example
```clojure
(defn do-something [x]
  "I take between 2 and 7 seconds to complete"
  (do
    (<!! (async/timeout 2000)) ;;wait i.e. do hard work
    (prn (str "I am processing..." (:id x))) ;; report status
    (<!! (async/timeout (rand-int 5000))) ;; more hard work
    (prn (str "I finished processing " (:id x))) ;;report status
    (assoc x :result (* 2 (:id x))))) ;; we return a :reult - id multiplied by 2
```
To do actual processing we could write something like this
```clojure
(defn dispatch [in]
  (let [out (chan)]
    (go (while true (>! out (do-something (<! in)))))
  out))
  
(-> 3 random-data-source dispatch echo)  
```    

Now assume our computatonal resources are unlimitted. 

50
```clojure
51
(defn do-something [x]
52
  "I take between 2 and 7 seconds to complete"
53
  (let [out (chan)]
54
    (go
55
      (<! (async/timeout 2000)) ;;wait i.e. do hard work
56
      (prn (str "I am processing..." (:id x))) ;; report status
57
      (<! (async/timeout (rand-int 5000))) ;; more hard work
58
      (prn (str "I finished processing " (:id x))) ;;report status
59
      (>! out (assoc x :result (* 2 (:id x)))))
60
    out))
61
```
62

But here is the catch.
