---
layout: post
title: Choose most narrow language feature possible, or why you should avoid reduce
---

A narrow language feature leaves little ambiguity to the intent of the code. Therefore, an experienced reader can selectively skip details of the expression without loss of understanding. In contrast, a reader is forced to carefully inspect an expression that doesn't use the narrowest feature to understand what the expression is intended to do. This principle is complementary to avoiding side-effects in your code and work best together.

The rest of this post will go over applications of choosing the narrowest language feature.

### Favor `partial` over the anonymous function reader macro `#(...)`. 

A common use of both `partial` and `#(...)` is to curry a function for use in a higher order function. 

Consider the following equivalent expressions that set up the configuration for a parameter search of a machine learning model. One uses `partial`, the other the anonymous function macro:

```Clojure
(map (partial generate-model-config default-config :slowdown-rate) 
     rates)

(map #(assoc generate-model-config default-config :slowdown-rate %) 
     rates)
```

When we use `(partial f a b c)`, we assure the reader that the inner function, `f` is called with parameters as listed followed by what ever additional parameters are passed to the resultant function of the `partial`. Because `partial` provides this assurance, the reader can safely glance at the inner function to understand the intent. In contrast, `#(...)`, provides no assurances; anything can be done within the anonymous function. Therefore, a reader must carefully look at the expression to know what the behavior is regardless of her expectation. 

A case in which you may prefer the anonymous function is if you want to constrain the number of arguments that are passed to the resultant function. With `partial`, the resultant function validly accepts an arbitrary number of inputs (including zero!) which may be problematic depending on the context. 

When the context itself constrains the number of inputs to the resultant function, such as when the resultant function is used by higher order functions, `partial` should be strictly preferred. 

When writing a function, take time to consider which parameters are most likely to be curried and put them first in the parameter list to allow for `partial` to be used. Many clojure core functions already do this. 

### Avoid `reduce` unless necessary

Most other higher order functions can be implemented in terms of `reduce`. This should hint to you that `reduce` is not narrow. The intent of an expression that uses `reduce` is captured in the reducing function. Therefore, the reader is forced to inspect the reducing function to determine what the expression does. 

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
   
```

Unfortunately, when using this function,  we get back a lot less data than we expect. 

If we want to understand why we are missing data, we need to read the reducing function carefully to find the point where a ride gets added to the list of rides in order to determine what is going wrong. In contrast, we can quickly scan past the `map` expressions in the second implementation because map provides the assurance it won't remove elements. We also know we should look closely at the `filter` expressions which provide the assurance it will only remove elements.  

You may want to use reduce if:
1. Shoe-horning your implementation into the narrower higher order functions is difficult or impossible
2. Performance is (actually) critical
 
 
### Other applications

1. `when` instead of `if`
2. Avoid `loop-recur` 
