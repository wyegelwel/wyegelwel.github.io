---
layout: post
title: Narrow language features; the case for avoiding reduce
---

A narrow language feature leaves little ambiguity to the intent of the code. Therefore, an experienced reader can selectively skip details of the expression without loss of understanding. In contrast, a reader is forced to carefully inspect an expression that doesn't use the narrowest feature to understand what the expression is intended to do. This principle is complementary to avoiding side-effects in your code and work best together.

The rest of this post will go over applications of choosing the narrowest language feature.

### Avoid `reduce` unless necessary

Most other higher order functions can be implemented in terms of `reduce`. This should hint to you that `reduce` is not narrow. 

The intent of an expression that uses `reduce` is captured in the reducing function. Therefore, the reader is forced to inspect the reducing function to determine what the expression does. 

Occasionally, `reduce` is used where a single narrower function could replace it. This tends to be rare because it is likely easier to write with the narrower function to begin with. For example, the below `reduce` expression can be swapped for a filter.

```Clojure
(reduce (fn [xs x] 
           (if (even? x)
              (conj xs x)
              xs))
    (range 100))

(filter even? (range 100))
```

More often, a `reduce` expression must be decomposed into multiple narrower higher order function calls combine with a threading macro. 
For example, suppose we have a function that accepts a raw dataset of cab rides and produces a cleaned version of the data. Below are two identical implementations of that function. One uses reduce, the other uses multiple narrow higher order functions. 

```Clojure
(defn clean-dataset
 [cab-rides]
 (reduce (fn clean-ride
          [rides ride]
          (let [ride (-> ride
                      (assoc :tip-percent 
                       (/ (:tip ride) (:total-fare ride)))
                      (update :pick-up-id location-id->location)
                      (update :destination-id location-id->location)
                      (rename {:pick-up-id :pick-up-location
                       :desination-id destination-location}))]
           (if (and (:destination-location ride)
                  (:pick-up-location ride))
            (conj rides ride)
            rides)))
  []
  cab-rides))

(defn clean-dataset
   [cab-rides]
   (->> cab-rides
        (map #(assoc % :tip-percent (/ (:tip %) (:total-fare %))))
        (map #(update % :pick-up-id location-id->location))
        (map #(update % :destination-id location-id->location))
        (map #(rename {:pick-up-id :pick-up-location
                       :destination-id :destination-location}))
        (filter :pick-up-location)
        (filter :destination-location))) 

;; A faster version of the threaded macro code above 
;; was recommended by Alex Miller (@puredanger)
;; This uses transducers which avoid making 
;; intermediate seqs between the map and filter calls
(defn clean-dataset
   [cab-rides]
   (into [] 
         (comp 
           (map #(assoc % :tip-percent (/ (:tip %) (:total-fare %))))
           (map #(update % :pick-up-id location-id->location))
           (map #(update % :destination-id location-id->location))
           (map #(rename {:pick-up-id :pick-up-location
                          :destination-id :destination-location}))
           (filter :pick-up-location)
           (filter :destination-location))
         cab-rides)
```

Unfortunately, when using this function,  we get back a lot less data than we expect. 

If we want to understand why we are missing data, we need to read the reducing function carefully to find the point where a ride gets added to the list of rides in order to determine what is going wrong. In contrast, we can quickly scan past the `map` expressions in the second implementation because map provides the assurance it won't remove elements. We also know we should look closely at the `filter` expressions which provide the assurance it will only remove elements.  

You may want to use reduce if:
1. Shoe-horning your implementation into the narrower higher order functions is difficult or impossible
2. Performance is (actually) critical

### Favor `partial` over the anonymous function reader macro `#(...)`. 

A common use of both `partial` and `#(...)` is to curry a function for use in a higher order function. 

Consider the following equivalent expressions that set up the configuration for a parameter search of a machine learning model. One uses `partial`, the other the anonymous function macro:

```Clojure
(->> rates
    (map #(generate-model-config default-config :slowdown-rate (inc %))))

(->> rates
     (map inc)
     (map (partial generate-model-config default-config :slowdown-rate)))
```

If you read the `#(...)` version carefully, you may have noticed the `inc` on the input whereas with partial this wasn't a problem. The reason is that when we use `(partial f a b c)`, we assure the reader that the inner function is simply being curried. Because `partial` provides this assurance, the reader can safely glance at the inner function to understand the intent. In contrast, `#(...)`, provides no assurances; anything can be done within the anonymous function. Therefore, a reader must carefully look at the expression to know what the behavior is regardless of her expectation. 

A case in which you may prefer the anonymous function is if you want to constrain the number of arguments that are passed to the resultant function. With `partial`, the resultant function validly accepts an arbitrary number of inputs (including zero!) which may be problematic depending on the context. 

When the context itself constrains the number of inputs to the resultant function, such as when the resultant function is used by higher order functions, `partial` should be strictly preferred. 

A second case to prefer `#(...)` is when the function is being applied many times and there are more than three parameters. The reason is that the implementation of `partial` switches to use `apply` with more than three parameters which drastically reduces performance.

When writing a function, take time to consider which parameters are most likely to be curried and put them first in the parameter list to allow for `partial` to be used. Many clojure core functions already do this. 


### Avoid `loop recur`

`loop` is similar to `reduce` in that it can be used to implement other higher order functions (including `reduce`!) but the intent of code that uses `loop` is even more muddled because you can do anything in a `loop` body. Often `loop recur` is used as a way to combine a complex reducing function (that should be split into multiple narrower functions) with a side effect. This is smell-y on two accounts, the first being the complex reducing function and the second being the side effect. 

One case in which `loop recur` does shine is in event-driven programming, generally with `async` channels. However, even here you should be careful to check that you aren't using `loop recur` to sneak in side effects where a transducer would otherwise fit.

Generally speaking, treat `loop recur` with suspicion.

### Conclusion

All of these cases come down to making it clear what the intent of the expression is without having to delve deep in to the expression. Favoring narrow features makes code easier to read and understand.

*The idea of reduce and loop recur being problematic language features came up during a conversation with Leif Walsh about Clojure. The idea of a narrow language feature is inspired by the use of narrow-ness to describe good names in Zach Tellman's Elements of Clojure*

*A big thank you to Leif Walsh, Shams Imam, Steven Radack, Daniel Posada, and Paul Schorfheide for their feedback while writing this!*

