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
We want to explore a little bit how core.async works and how we can use it to deal with random events and parallel processing of data streams.
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
Usually we want to process incomming data. Assume we have a function which takes longer than the intervals between the arriving of new data and whose running time may vary. We can emulate this with the following example
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
We want to apply it to data comming from random-data-source. We could write something like this
```clojure
(defn dispatch [in]
  (let [out (chan)]
    (go (while true (>! out (do-something (<! in)))))
  out))
  
(-> 3 random-data-source dispatch echo)      
```
As expected the output is
```
"I am processing 0..."
"I finished processing 0"
"#echo: {:result 0, :id 0}"
"I am processing 1..."
"I finished processing 1"
"#echo: {:result 2, :id 1}"
"I am processing 2..."
"I finished processing 2"
"#echo: {:result 4, :id 2}"
```
Clearly the processing is done sequentially. Now let's add a bit more complexity. Assume that we want to reduce all the processed data. Here is a dummy example which will compute the sum of all results.
```clojure
(defn reducer [in]
  (let [out (chan)]
    (go-loop [acc {:result_id 0 :value 0}]
      (do
        (let [x (:result (<! in))
              y (update acc :value #(+ %1 x))] ;; temporary var y to hold the new result
          (>! out y)
          (recur (update y :result_id inc))))) ;;inc result_id for the next iteration
    out))
```
Now we can connect all four functions
```clojure
(-> 3 random-data-source dispatch reducer echo)      
```
This produces the following output
```
"I am processing 0..."
"I finished processing 0"
"#echo: {:result_id 0, :value 0}"
"I am processing 1..."
"I finished processing 1"
"#echo: {:result_id 1, :value 2}"
"I am processing 2..."
"I finished processing 2"
"#echo: {:result_id 2, :value 6}"
```
While it is fine, we have two weak points. First, the code isn't beautiful. For example there is only one line in reducer which is really important for our application (namely the one where y is computed, all others are not specific to our application). And the second is that the processing is done in a serial way. It would be much more intersting if we needn't to wait until do-something finishes and could start processing new message immidiately after it becomes available from random-data-source. Let's pretend that our computational resources (CPU cores and RAM) are infinite and we can run as much parallel tasks as we want. 

Conceptually we want to change the dispatch function so that the call to do-something becomes asynchroneuse. We can modify do-something so that it would return a channel immidiately. Than the only problem would be that every call to do-something will produce a new channel and we have to route all data somehow to the reducer function. For this core.async has a mechanism called channel mix. Let's implement this strategy.

First, we have to modify do-something. The new version could look like this
```clojure
(defn do-something-chan [x]
  "I take between 2 and 7 seconds to complete"
  (let [out (chan)]
    (go
      (<!! (async/timeout 2000)) ;;wait i.e. do hard work
      (prn (str "I am processing " (:id x) "...")) ;; report status
      (<!! (async/timeout (rand-int 5000))) ;; more hard work
      (prn (str "I finished processing " (:id x))) ;;report status
      (>! out (assoc x :result (* 2 (:id x))))) ;; we return a :reult - id multiplied by 2
    out))
```
and then to adjust the dispatch. Its new version is 
```clojure
(defn dispatch-multi [in]
  (let [out (chan)
        mix (async/mix out)] ;; create channel mix
    (go (while true
          (async/admix mix (do-something-chan (<! in)))))
    out))
```
Finally we can run our new cahain
```clojure
(-> 4 random-data-source dispatch-multi reducer echo)  
```
This time the output is
```
"I am processing 0..."
"I finished processing 0"
"#echo: {:result_id 0, :value 0}"
"I am processing 1..."
"I am processing 2..."
"I am processing 3..."
"I finished processing 2"
"#echo: {:result_id 1, :value 4}"
"I finished processing 1"
"#echo: {:result_id 2, :value 6}"
"I finished processing 3"
"#echo: {:result_id 3, :value 12}"
```
The processing runs parallely! We can see that all four chunks of data are started to being processed as they arrive and that in this case "finished processing 2" comes before "finished processing 1". Note that the sequence will differ every time we run the code, but not the end result which is the last line. 

We see a common pattern
```
<function body> -
-> (let [out (chan)] <function body> out)
```
It applies to all our functions, with the exception of dispatch-mix which adds a channel mixer into game. So we could write a macro which
does this for us.




