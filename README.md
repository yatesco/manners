# manners [![Build Status](https://travis-ci.org/RyanMcG/manners.png?branch=master)](https://travis-ci.org/RyanMcG/manners)

A validation library built on using predicates properly.

```clojure
;; Add the following to dependencies in your project.clj
[manners "0.4.0"]
```

## Thoughts

This library is the result of reading [Functional JavaScript][] and being inspired by the simplicity of the validation functions [the author][fogus] creates.
Composing predicates is a very easy thing to do in Clojure and I wanted a library which takes advantage of that [without too much magic](#comparisons).

In older versions of manners there was a modern dialect as well as the default, victorian-style function names.
This has been removed for greater consistency.

## Usage

### Terminology

First some terms vital to manner's lexicon.

* etiquette - A sequence of manners
* manner - One or more coaches or predicate message pairs
* coach - A function that takes a value and returns a sequence (preferably lazy) of messages.
  A message could be anything but is most often a string describing what makes the given value invalid.
  Coach functions (not vars) must have the meta `^:coach`.

### Creating coaches

#### `manner`

There are several functions which create coaches.
The most essential is `manner` which creates a coach from a manner like so:

```clojure
(def div-by-six-coach (manner even? "must be even"
                              #(zero? (mod % 6)) "must be divisible by 6"))
(div-by-six-coach 1) ; => ("must be even")
(div-by-six-coach 2) ; => ("must be divisible by 6")
(div-by-six-coach 12) ; => ()
```

`manner` is an idempotent function.

```clojure
(def div-by-six-coach2 (manner (manner (manner (manner div-by-six-coach)))))
;; The behaviour of div-by-six-coach and div-by-six-coach2 is the same
(div-by-six-coach2 2) ; => ("must be divisible by 6")
```

#### `manners`

`manners` creates a coach from a one or more manners or coaches.
Instead of returning the first matching message it returns the results of every coach.

```clojure
(def div-by-six-and-gt-19-coach
  (manners div-by-six-coach
           [#(>= % 19) "must be greater than or equal to 19"]))

(div-by-six-and-gt-19-coach 1)
; => ("must be even" "must be greater than or equal to 19")
(div-by-six-and-gt-19-coach 2)
; => ("must be divisible by 6" "must be greater than or equal to 19")
(div-by-six-and-gt-19-coach 12)
; => ("must be greater than or equal to 19")
(div-by-six-and-gt-19-coach 24) ; => ()
```

`manners` is also idempotent.

```clojure
(def div-by-six-coach2 (manners (manner (manners (manners div-by-six-coach)))))
;; The behaviour of div-by-six-coach and div-by-six-coach2 is the same
(div-by-six-coach2 2) ; => ("must be divisible by 6")
```

In fact, manners and manner can be interchanged if the only argument is a single coach.

However, they will behave differently with multiple coaches.

#### `etiquette`

This function is almost identical to manners.
Actually, manners is defined using etiquette.

```clojure
(defn manners [& etq]
  (etiquette etq))
```

Instead of passing in an arbitrary number of arguments, etiquette is a [unary][] function which takes all manners as a sequence.

```clojure
;; These are all the same...
(def div-by-six-and-gt-19-coach (etiquette [[div-by-six-coach]
                                            [#(>= % 19) "must be greater than or equal to 19"]])
(def div-by-six-and-gt-19-coach (etiquette [div-by-six-coach
                                            [#(>= % 19) "must be greater than or equal to 19"]])
(def div-by-six-and-gt-19-coach (manners [div-by-six-coach]
                                         [#(>= % 19) "must be greater than or equal to 19"])
(def div-by-six-and-gt-19-coach (manners div-by-six-coach
                                         [#(>= % 19) "must be greater than or equal to 19"])
```

### Helpers

#### `bad-manners` &amp; `coach`

```clojure
(use 'manners.victorian)
(def etq [[even? "must be even"
           #(zero? (mod % 6)) "must be divisible by 6"]
          [#(>= % 19) "must be greater than or equal to 19"]])
(bad-manners etq 11)
; => ("must be even" "must be greater than or equal to 19")
(bad-manners etq 10) ; => ("must be greater than or equal to 19")
(bad-manners etq 19) ; => ("must be even")
(bad-manners etq 20) ; => ("must be divisible by 6")
(bad-manners etq 24) ; => ()
```

`bad-manners` is simply defined as:
```clojure
(defn bad-manners
  [etq value]
  ((etiquette etq) value))
```

Memoization is used so that subsequent calls to `coach` and the function
generated by `coach` does not repeat any work. That also means predicates used
in an etiquette should be referentially transparent.

#### `proper?` &amp; `rude?`

Next are `proper?` and `rude?`.  They are complements of each other.

```clojure
;; continuing with the etiquette defined above.
(proper? etq 19) ; => false
(proper? etq 24) ; => true
(rude? etq 19)   ; => true
(rude? etq 24)   ; => false
```

`proper?` is defined by calling `empty?` on the result of `bad-manners`. With
the memoization you can call `proper?` then check `bad-manners` without doubling
the work.

```clojure
(if (proper? etq some-value)
  (success-func)
  (failure-func (bad-manners etq some-value))
```

Of course we all want to be dry so you could do the same as above with a bit
more work that does not rely on the memoization. Pick your poison.

```clojure
(let [bad-stuff (bad-manners etq some-value)]
  (if (empty? bad-stuff)
    (success-func)
    (failure-func bad-stuff)))
```

#### `avow!` &amp; `falter`

Next on the list is `avow!` which takes the results of a call to `bad-manners`
and throws an `AssertionError` when a non-empty sequence is returned.
Internally `avow!` is conceptionally like the composition of `falter` (which
does the throwing) and an etiquette coach.

```clojure
;; assume `etq` is a valid etiquette and `value` is the value have
;; predicates applied to.
((comp falter (etiquette etq)) value)
```

#### `defmannerisms`

The last part of the API is `defmannerisms`. This is a helper macro for defining
functions that wrap the core API and a given etiquette.

```clojure
(defmannerisms empty-coll
 [[identity "must be truthy"
   coll? "must be a collection"]
  [empty? "must be empty"]])

(proper-empty-coll? []) ; => true
(rude-empty-coll? []) ; => false
(bad-empty-coll-manners nil) ; => ("must be truthy")
(bad-empty-coll-manners "") ; => ("must be a collection")
(bad-empty-coll-manners "a") ; => ("must be a collection" "must be empty")
(avow-empty-coll! 1)
; throws and AssertionError with the message:
;   Invalid empty-coll: must be truthy

;; And so on.
```

### Composability

With version 0.3.0, etiquettes can now contain coaches.

The abstraction made to accomplish this was to make a matter a sequence of
coaches by converting predicate message pairs into coaches.
You probably do not care about that though so just look at the example below.

```clojure
(def my-map-coach (etiquette [[:a "must have key a"
                               (comp number? :a) "value at a must be a number"]
                              [:b "must have key b"]]))
;; Just for reference
(my-map-coach {}) ; => ("must have key a" "must have key b")
(my-map-coach {:a 1}) ; => ("must have key b")
(my-map-coach {:a true}) ; => ("value at a must be a number" "must have key b")
(my-map-coach {:b 1}) ; => ("must have key a")

;; We can copy a coach by making it the only coach in a one manner etiquette.
(def same-map-coach (etiquette [[my-map-coach]]))
;; Or with manners
(def same-map-coach (manners [my-map-coach]))
(def same-map-coach (manners my-map-coach)) ;; This works too
;; Or with manner
(def same-map-coach (manner my-map-coach))
(same-map-coach {}) ; => ("must have key a" "must have key b")
(same-map-coach {:a 1}) ; => ("must have key b")
(same-map-coach {:a true}) ; => ("value at a must be a number" "must have key b")
(same-map-coach {:b 1}) ; => ("must have key a")

;; We can also add on to a coach.
(def improved-map-coach
 (manners [my-map-coach (comp vector? :b) "value at b must be a vector"]
          ;; If the entirety of my-map-coach passes our additional check on b's
          ;; value will take place

          ;; We can also add more parallel checks
          [:c "must have key c"
           (comp string? :c) "value at c must be a string"]))

(improved-map-coach {}) ; => ("must have key a" "must have key b" "must have key c")
(improved-map-coach {:a 1}) ; => ("must have key b" "must have key c")
(improved-map-coach {:a true}) ; => ("value at a must be a number" "must have key b" "must have key c")
(improved-map-coach {:a true :b 1}) ; => ("value at a must be a number" "must have key c")
(improved-map-coach {:a 1 :b 1}) ; => ("value at b must be a vector" "must have key c")
(improved-map-coach {:a 1 :b [] :c "yo"}) ; => ()
```

### With

To avoid having to consistently pass in etiquette as a first argument you can
use the `with-etiquette` macro.

```clojure
(use 'manners.with)
(with-etiquette [[even? "must be even"
                  #(zero? (mod % 6)) "must be divisible by 6"]
                [#(>= % 19) "must be greater than or equal to 19"]]
  (proper? 10) ; => false
  (invalid? 11) ; => true
  (errors 19) ; => ("must be even")
  (bad-manners 20) ; => ("must be divisible by 6")
  (bad-manners 24)) ; => ()
```

## Comparisons

Clojure already has quite a few good validation libraries. This library is not
greatly different and has no groundbreaking features. However, it does differ a
couple of key ways.

*manners*
* is small and simple.
* uses memoization for improved performance.
* works on arbitrary values, not just maps.

The following are some descriptions of other validation
libraries in the wild. They are listed alphabetically.

* [*bouncer*](https://github.com/leonardoborges/bouncer) is a neat DSL for
  validating maps which works particularly well on nested maps. It accepts
  predicates as validators and even checks custom meta on those predicates for
  special behaviour like message formatting. A common trend in other validation
  libraries, it returns a custom formated data structure:

  > `validate` takes a map and one or more validation forms and returns a
  > vector.
  >
  > The first element in this vector contains a map of the error messages,
  > whereas the second element contains the original map, augmented with the
  > error messages.

* [*metis*](https://github.com/mylesmegyesi/metis) is a keyword based DSL for
  validating maps by creating higher order functions. It returns a map reusing
  the validated keys with values being a sequence of errors. It advertises
  validator composition which works great for validating nested maps.
* [*mississippi*](https://github.com/mikejones/mississippi) offers a lot of the
  same functionality by returning maps containing errors on each key. Of course
  that means this library is useful for validating maps only.
* [*Red Tape*](http://sjl.bitbucket.org/red-tape/) is a form validation library.
  It is not meant to be general purpose at all in an effort to "reduce friction"
  in its problem domain.
* [*sandbar*](https://github.com/brentonashworth/sandbar) is a web application
  library which, along with many other neat features, offers form generation
  with built in validation. Obviously this is also very focused on its singular
  use case.
* [*validateur*](https://github.com/michaelklishin/validateur) a popular library
  with good documentation which does much the same thing as *mississippi*.
* [*valip*](https://github.com/weavejester/valip) is perhaps the must similar of
  the validation libraries to *manners* in that it is based on the simple
  application of predicates. However, its use does not differ greatly from
  *validateur* or *mississippi*. It also provides [a helpful suite of predicates
  and higher-order, predicate generating functions][valip.predicates] which are
  compatible with *manners* (since they are just predicates).
* [*vlad*](https://github.com/logaan/vlad) is another general purpose validation
  library.

  > Vlad is an attempt at providing convenient and simple validations. Vlad is
  > purely functional and makes no assumptions about your data. It can be used
  > for validating html form data just as well as it can be used to validate
  > your csv about cats.

  I think its greatest strength is [its composition
  abilities](https://github.com/logaan/vlad#composition). Etiquettes in
  *manners* are just vectors so they can easily be constructed together. Vlad's
  `chain` method stops checking after the first failed validator. I have since
  added a feature to *manners* to provide similar support (multiple predicate
  message pairs in a single manner).

Clearly validating maps is a common problem in Clojure. A common use case is the
web application which needs to validate its parameters. Another is custom maps
without a strongly defined (i.e. typed) schema.

Although it may be less common I find there are cases where validating arbitrary
values is very useful. None of the above work with non-maps nor could they be
easily modified to do so because they are, by design meant for keyed data
structures.

Having been primarily doing Rails development for my, so far, short professional
career I've become accustomed to keyed errors but I have rarely found much value
in them. When I want to flash a message to a user I desire a well worded message
for the given context and `Model#errors.full_messages` has never been it. A
simple sequence of messages is almost always preferable.

In an effort to be more generic this library differs from those above by not
caring on what field an error occurred. There is no concept of a value needing
to have fields at all. Another consequence of not requiring fields is that
validating related fields is easier. Consider the following etiquette and data.

```clojure
(def data {:count 3 :words ["big" "bad" "wolf"]})
(defn count-words-equal? [{cnt :count words :words}]
  (= cnt (count words)))
(def etq
  [[(comp number? :count) "count must be a number"]
   [count-words-equal? "count must equal the length of the words sequence"]])
```

This etiquette works fine with *manners*. Other libraries make validating
relationships between fields much more difficult by limiting a predicate's
application to the value of the field it is keyed to. The benefit of doing it
that way is you can concisely define per field validations. The alternative,
having to drill down to the field you mean to apply your predicate to, may seem
like more work but it is still quite concise when using `comp` (see above
example).

## Test

Vanilla and delicious `clojure.test`.

```bash
lein test
```

## License

Copyright © 2014 Ryan McGowan

Distributed under the Eclipse Public License, the same as Clojure.

[unary]: http://en.wikipedia.org/wiki/Arity
[Functional JavaScript]: http://www.amazon.com/Functional-JavaScript-Introducing-Programming-Underscore-js/dp/1449360726
[fogus]: http://fogus.me/
[valip]: https://github.com/weavejester/valip
[valip.predicates]: https://github.com/weavejester/valip/blob/master/src/valip/predicates.clj
[doc]: http://www.ryanmcg.com/manners
