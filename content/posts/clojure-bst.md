---
title: "Binary Search Tree in Clojure"
date: 2018-11-04T16:04:56-04:00
draft: true
tags: ["clojure"]
menu:
  main:
    parent: tutorials
---



Structure of the BST

```clojure
Empty tree - nil
Single node - [5 nil nil]
Three nodes - [5 [3 nil nil] [7 nil nil]]
```

Insert into BST:
```clojure
(defn insert [tree & values]
  (cond
    (empty? values) tree
    (nil? tree) (apply insert [(first values) nil nil] (rest values)) 
    (< (first values) (first tree))
        (apply insert [(first tree) (insert (second tree) (first values)) (last tree)] (rest values))
    :else (apply insert [(first tree) (second tree) (insert (last tree) (first values))] (rest values))
  )
)
```

Depth of a BST:

```clojure
(defn depth [tree]
  (if (nil? tree) 0
  (inc (apply max (map depth tree))))
) 
```
