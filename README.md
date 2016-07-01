# clojureme

This repository contain notes about my observations about Clojure and Clojurescript and sinppets of code which I found useful while learning to write Clojure. It is primary intended for myself to keep track of all my findings about this fancy and fascinating language.

## 1 Generating tables with Enfocus

```clojure
(defn make-table-row! [params]
      (let [values (select-keys (:data params) (map name (:which params)))]
          (ef/html [:tr {:class "row"}
                    (map (fn [x]
                            [:td {:data-class (key x) } (->> x second str)]) values)])))


(defn populate-table! [target columns data]
  (do
    (loop [items data n 1]
      (when-some [item (first items)]
        (do
          (ef/at target (ef/append (make-table-row {:data item :which columns})))
          (recur (rest items) (inc n)))))))
``` 

