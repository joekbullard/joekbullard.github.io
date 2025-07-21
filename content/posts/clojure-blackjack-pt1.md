---
title: "Blackjack with Clojure - Part 1"
date: 2025-07-20T21:10:11+01:00
draft: false
tags: ['clojure']
---

One of the first small projects I like to do when learning a new language is to model a deck of cards. It's a good way to familiarise yourself with the basics of language, such as data structures and flow control, once you have a functioning card deck you can take it a step further and model a game like poker or blackjack. As part of my Clojure learning, I have been playing around a bit and thought it would be fun to model the card game blackjack.

First things first, you want something to represent a card. In clojure the easiest way to do this is to make a simple map like so:

```clojure
(defn make-card [rank suit]
  {:rank rank :suit suit})
```

Not much to this really, like any clojure map you can access the values using the keywords directly:

```clojure
(make-card ("A" "Spaces"))
;; {:rank "A", :suit "Spades"}

(:suit (make-card "A" "Spades"))
;; "Spades"

(:rank (make-card "A" "Spades"))
;; "Ace"
```

Now that there is a card implemented, the next step is to figure out a way to build a deck. Again, sticking to idiomatic clojure makes this simple.

```clojure
(def ranks ["2" "3" "4" "5" "6" "7" "8" "9" "10" "J" "Q" "K" "A"])
(def suits ["Diamonds" "Hearts" "Spades" "Clubs"])

(defn make-deck [ranks suits]
  (for [r ranks s suits] (make-card r s)))
```

First we made two vectors containing ranks and suits. Next the `make-deck` function uses a nested `for` loop to generate all combinatios of ranks and suits. The `for` syntax might appear somewhat unusual if you're not familiar to clojure. In essence, with each outer iteration `(for [r ranks s suits ...` is binding each element from `ranks` to `r` then within that, each element of `suits` is bound to `s`. So it loops through every combination of rank and suit, with the `make-deck` function called on each, returning a sequence of cards. We can test this out by calling the function and insepcting the first few cards:

```clojure
(def deck (make-deck ranks suits))

(first deck)
;; {:rank "2", :suit "Diamonds"}
(second deck)
;; {:rank "2", :suit "Hearts"}
```

So we have cards and we have a deck. One more useful functonality is to get the value of the card, you will have noticed the `:rank` for the card map is a string and has no numerical value, so they are currently not much use for most card games where card value is relevant. We are wanting to model blackjack, so in addition to the numbered cards, J, Q and K have a value of 10 and A has a value of 11 (or 1 depending on game context, but I'll ignore this for now).

```clojure
(defn get-value [card]
  (let [rank (:rank card)]
    (cond
      (#{"J" "Q" "K"} rank) 10
      (= rank "A") 11
      :else (Integer. rank))))
```

This functon uses a simple  `cond` statement - whichever condition first evaluates to true is returned as the value. The `:else` block casts the rank to an integer if the card rank is numbered, else you get 10 for the face cards and 11 for an ace - we can try this out with a few examples:

```clojure
(get-value (make-card "8" "Spades")) ;; 8
(get-value (make-card "A" "Spades")) ;; 11
(get-value (make-card "J" "Spades")) ;; 10
```

So we can generate a deck of cards with a suit and rank, and from that we can get a face value. These are the basic builing blocks of the model. In the next post I will play around with flow control and immutable data structures to build a blackjack game.