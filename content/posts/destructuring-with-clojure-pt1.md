---
title: "TIL how to destructure with clojure"
date: 2025-06-08T21:17:51+01:00
draft: false
tags: ['til', 'clojure']
---

This isn't strictly a "TIL" but it seemed like a good place to start as I've recently started learning Clojure for a work project. Compared to a language like Python the syntax is radically different, most code written in Clojure comes in what is known as a form - a statement contained within parentheses, for example:

```clojure
(+ 2 (* 2 2)) 
;; 6
```

One aspect I've found particularly challenging is the syntax around destructuring of maps and vectors within a `let` form. `let` is what is known as a special form and is used to bind a data structure to a symbol that is usable within the scope of the `let` form. For example:

```clojure
(let [x 2]
    (+ x (* x x)))
;; 6
```

Here the value 2 is bound to the variable `x` while in the `let` form which is then used to repeat the previous calculation. While this a fairly straightforward example, it can get much harder when working with multiple `let` forms or using the different destructuring patterns requried by different data structures.

Here is an example using a vector:

```clojure
(def twos [2 2 2])

(let [[x y z] twos]
    (+ x (* y z)))
;; 6
```

In this example - the vector `[2 2 2]` is assigned to the variable `example`. The indiviual vector values are then destructed sequentially and bound to the local symbols  `[x y z]` respectively within the scope of the `let` form, and once again the same calculation is run. Again, not massively complex, but somewhat different to Python.

And here is an example using a map, with the values destructured associatively:

```clojure
(def numbers {:one 1
              :two 2
              :three 3})

(let [{a :one b :two c :three} numbers]
    (+ a b c))
;; 6
```

One aspect I've found useful to remember when using associative destructuring, you will always need to have an even number of key value pairs.

You can also combine both:

```clojure
(def numbers {:one 1
              :two 2
              :threefour [3 4]})

(let [{a :one b :two [c d] :threefour} numbers]
    (+ a b c d))
;; 10
```

Here we are using a map data structure with integer values bound to key symbols - denoted by `:`. This is then destuctured associatively within the `let` statement. Note also that values 3 and 4 are nested, so these values are destructured sequentially. While initially finding it a little hard to comprehend, it has quickly become one of the many things I've come to appreciate about clojure.
