---
title: "TIL about threading in Clojure"
date: 2025-06-10T21:23:28+01:00
draft: false
tags: ['clojure', 'til']
---

Clojure has an incredble range of functions and macros in its core library - [just look at them all!](https://clojuredocs.org/quickref) While I don't expect to ever learn them all by heart, there are some that I've figured are important for shaping how you breakdown a problem with Clojure. I've particularly enjoyed improving my understanding of the threading macros `->` and `->>` (There are others, such as `some->`, but I'm yet to use them myself).

Threading is not really a thing in Python and it took me a bit of time to really understand it's value in Clojure. In essence, threading in Clojure allows you to chain commands together - with the output of one form feeding into the next. Consider the following set of forms:

```clojure
(update (assoc (assoc {:a 2} :b 4) :c 5) :a inc)
;; {:a 3, :b 4, :c 5}
```

It's not the easiest set of forms to parse. An alternative way to do this using a threading macro would be the following:

```clojure
(-> {:a 2}
    (assoc :b 4)
    (assoc :c 5)
    (update :a inc))
;; {:a 3, :b 4, :c 5}
```

As you can see, the result is the same, just rewritten in a top down way that's easier to parse, with the output from one line feeding into the next. When it comes to compile time, the `->` macro actually rewrites it to match the first version. It's worth noting that anything you can do using threading can also be done using nested forms, but sometimes making it more readable is important.