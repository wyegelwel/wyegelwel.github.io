---
layout: post
title: Error Handling with Clojure Async
---


If you are reading this, there is a good chance an Exception was thrown in one of your async blocks and something *unexpected* happened. If you are using at least Clojure 1.6.0, your exception printed to stderr and the go block stopped execution. If you are using a version of clojure before 1.6.0, your exception disappeared into the ether. Either case is less than ideal. The problem is that async blocks run in a different thread than they were called from and therefore you can't catch exceptions outside of the block. In code this looks like:

```Clojure
(require '[clojure.core.async :refer :all])

;; Clojure 1.6.0 and above, this exception stack trace prints to stderr
;; Less than Clojure 1.6.0, exception is dropped, and nil is returned
(<!! (go (throw (Exception.))))

;; You will never see "Exception caught" in stdout
(try
   (go (throw (Exception.)))
   (catch Exception e 
   		(println "Exception caught")))
```

The crux of the problem is that async blocks run in a thread pool and the pool tries to do something reasonable with the exception for all cases. Specifically, it makes use of java's [DefaultUncaughtExceptionHandler](http://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler) which seem to have different behavior in Clojure 1.6.0 vs Clojure 1.5.0. Don't worry though, all hope is not lost! You have a few options available to you to handle these exceptions with care.

### Default Uncaught Exception Handler 

The first way to handle errors with async is just to define your own default uncaught exception handler. This is convenient, especially to patch async code that is already written, because you don't need to touch the existing code. You can simply add a block to the top of your file and your error handling works. 

```Clojure
(require '[clojure.core.async :refer :all])

(Thread/setDefaultUncaughtExceptionHandler
	(reify Thread$UncaughtExceptionHandler
		(uncaughtException [_ thread throwable]
			(println (.getMessage throwable))
			(System/exit 1))))

(go (throw (Exception. "OOOOOOOOOOOOHHHH NNOOOOOOOOO!!")))

;prints OOOOOOOOOOOOHHHH NNOOOOOOOOO!!, then exits
```

The problem with this method is that it is a catch all for every async block and therefore if you want to have different error handling based on different exceptions you will either have to make a fairly complex handler or use the methods below. Additionally, be aware that the UncaughtExceptionHandler may be set far from your async blocks (hopefully not in other libraries!) and therefore may cause confusion when trying to debug. Finally, your exception has been removed from the standard code paths of your program which limits the actions you can take.

### Exception channels

The second method is to use designated error channels for exceptions. Once an exception occurs, your async block will catch it, and put it on an *exception channel*. That channel is constantly pulled from and the exception is handled. This method is similar to setting a default uncaught exception handler in that the exception is removed from the normal flow of your program, but provides you additionally flexibility by allowing you have multiple exception channel and handler pairs. 

```Clojure
(require '[clojure.core.async :refer :all])

(def ex-chan (chan))

;; Exception handler
(go
  (let [e (<! ex-chan)] 
    (println (.getMessage e))
    (System/exit 1)))

(go 
  (try 
    (throw (Exception. "Not again!!"))
  (catch Exception e
    (>! ex-chan e))))

; prints Not again! and exits
```

### The `<?` construct

The final method involves using the channel from the async block to transmit errors. This method is covered by [David Nolan](http://swannodette.github.io/2013/08/31/asynchronous-error-handling/) and [Martin Trojer](http://martintrojer.github.io/clojure/2014/03/09/working-with-coreasync-exceptions-in-go-blocks/) and provides the most flexibility. Nolan introduces a new async primitive `<?` which will pull off a channel and throw an exception if the item pulled off is throwable. This mirrors the contract of a future in terms of exception handling, namely, the exception is thrown once the value is read. Using this method, you can replicate error handling as if you were running a single threaded application. Be aware that depending on how you write your async code, this may be difficult to fit in.  

```clojure
(require '[clojure.core.async :refer :all])

;; Async pop that throws an exception if item returned is throwable
;; This function comes from David Nolen 
(defn <?? [c] 
	(let [returned (<!! c)]
		(if (instance? Throwable returned)
			(throw returned)
			returned)))

(<?? (go 
       (try
         (throw (Exception. "This is exceptional!"))
       (catch Exception e
          e)))

; This throws the exception as we expect!
```

### What to choose?

All three of these methods are viable, and are by no means mutually exclusive. In fact, it is recommended to set a default uncaught exception handler if for nothing else to let you know there were uncaught exceptions that *should* be caught. However, you will likely get the most mileage out of the `<?` pattern.  
