---
title: "Blackjack with Clojure - Part 2"
date: 2025-07-23T21:44:28+01:00
draft: false
tags: ['clojure']
---

This is the second of two posts using Clojure to make a simple blackjack program, in the [previous post]({{< relref "clojure-blackjack-pt1" >}}) we made a hand, deck and a way to get the face value of the card. We will build on this to include scoring and the actual game logic.

Building on where we finished last time, getting the value of individual cards, we will make a function to get the score of multiple cards. The easiest way to do this is to use our `get-value` function with `map` and then `reduce` to get the combined total:

```clojure
(def first-hand [(card "7" "Spades") (card "6" "Spades")])

(defn hand-value  [h]
  (reduce + (map #(get-value %) h)))
  
(hand-value first-hand);; 13
```

This gives the correct value. However, there is a problem:

```clojure
(def second-hand [(card "A" "Spades") (card "A" "Spades")])

(hand-value second-hand);; 22
```

If we get two Aces we get 22, which means bust. In blackjack Aces can be either 1 or 11, so if you draw two you don't go bust straight away. Therefore, there needs to be additional logic to account for having Aces in a hand. There are a few ways to do this, I'm going to use `loop` with `recur` to adjust the value of Aces,

```clojure
(defn hand-value
  [hand]
  (loop [value (apply + (map get-value hand))
         aces (count (filter #(= (:rank %) "A") hand))]
    (if (or (<= value 21) (= aces 0))
      value
      (recur (- value 10) (dec aces)))))

(hand-value [(card "A" "Spades") (card "A" "Diamonds")]);; 12

(hand-value [(card "K" "Spades") (card "K" "Diamonds") (card "K" "Clubs")]);; 30

```

Let's break down what's happending here. Using hand-value takes a hand - assumed to be one or more of the card maps we defined previously. The initial hand value is bound to `value` and the number of Ace cards to `aces`. From here there is an if statement, if either `value` is below 21 or the number of `aces` is equal to zero then the hand value will be returned. If not, and there are `aces` left, the score and number of aces will be reduced. This means if you have a hand of two aces like in the example, the first time `value` will be 22 with `aces` count of two. The second time around the `value` will be reduced by 10 along with the number of `aces`, this takes the hand value below the threshold of 21, meaning the if statement returns true, returning the value. In the second example wtih a hand of three Kings, the inital value is over 21 but there are no aces, so the if statement returns true and the value of 30 is returned.

We can clean this up a bit by adding a predicate function that returns `true` if a card is an ace:

```clojure
(defn ace? [c]
  (= (:rank c) "A"))

  (defn hand-value
  [hand]
  (loop [value (apply + (map get-value hand))
         aces (count (filter ace? hand))]
    (if (or (<= value 21) (= aces 0))
      value
      (recur (- value 10) (dec aces)))))
```

While we're at it, we can also make one for face cards and make our `get-value` function a little cleaner.

```clojure
(defn face? [c] 
  (#{"J" "Q" "K"} (:rank c)))

(defn get-value [c]
  (cond
    (face? c) 10
    (ace? c) 11
    :else (Integer. (:rank c))))
```

This doesn't change much but makes the code a little cleaner and more consistent.

That's it for this post, in the next one we will get around to dealing cards and playing a full game.