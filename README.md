# Expound

Expound formats `clojure.spec` error messages in a way that is optimized for humans to read.

Expound is in alpha while `clojure.spec` is in alpha.

## Usage

[![Clojars Project](https://img.shields.io/clojars/v/expound.svg)](https://clojars.org/expound)

### `expound`

Replace calls to `clojure.spec.alpha/explain` with `expound.alpha/expound` and to `clojure.spec.alpha/explain-str` with `expound.alpha/expound-str`.

```clojure
(require '[clojure.spec.alpha :as s])
;; for clojurescript:
;; (require '[cljs.spec.alpha :as s])
(require '[expound.alpha :as expound])

(s/def :example.place/city string?)
(s/def :example.place/state string?)
(s/def :example/place (s/keys :req-un [:example.place/city :example.place/state]))

(s/explain :example/place {})
;; val: {} fails spec: :example/place predicate: (contains? % :city)
;; val: {} fails spec: :example/place predicate: (contains? % :state)
;; :clojure.spec.alpha/spec  :example/place
;; :clojure.spec.alpha/value  {}

(expound/expound :example/place {})
;; -- Spec failed --------------------

;;   {}

;; should contain keys: `:city`,`:state`

;; -- Relevant specs -------

;; :example/place:
;;   (clojure.spec.alpha/keys
;;    :req-un
;;    [:example.place/city :example.place/state])

;; -------------------------
;; Detected 1 error

(s/explain :example/place {:city "Denver", :state :CO})
;; In: [:state] val: :CO fails spec: :example.place/state at: [:state] predicate: string?
;; :clojure.spec.alpha/spec  :example/place
;; :clojure.spec.alpha/value  {:city "Denver", :state :CO}

(expound/expound :example/place {:city "Denver", :state :CO})
;; -- Spec failed --------------------

;;   {:city ..., :state :CO}
;;                      ^^^

;; should satisfy

;;   string?

;; -- Relevant specs -------

;; :example.place/state:
;;   clojure.core/string?
;; :example/place:
;;   (clojure.spec.alpha/keys
;;    :req-un
;;    [:example.place/city :example.place/state])

;; -------------------------
;; Detected 1 error
```

### `*explain-out*`

To use other Spec functions, set `clojure.spec.alpha/*explain-out*` (or `cljs.spec.alpha/*explain-out*` for ClojureScript) to `expound/printer`.

(Setting `*explain-out*` does not work correctly in ClojureScript versions prior to `1.9.562` due to differences in `explain-data`)

```clojure
(require '[clojure.spec.alpha :as s])
;; for clojurescript:
;; (require '[cljs.spec.alpha :as s])
(require '[expound.alpha :as expound])

(s/def :example.place/city string?)
(s/def :example.place/state string?)

;;  Use `assert`
(s/check-asserts true) ; enable asserts

;; Set var in the scope of 'binding'
(binding [s/*explain-out* expound/printer]
  (s/assert :example.place/city 1))

;; Or set it for the current thread.
(set! s/*explain-out* expound/printer)
(s/assert :example.place/city 1)

;; Use `instrument`
(require '[clojure.spec.test.alpha :as st])

(s/fdef pr-loc :args (s/cat :city :example.place/city
                            :state :example.place/state))
(defn pr-loc [city state]
  (str city ", " state))

(st/instrument `pr-loc)
(pr-loc "denver" :CO)

;; You can use `explain` without converting to expound
(s/explain :example.place/city 123)
```

If you are enabling Expound in a non-REPL environment, remember that `set!` will only change `s/*explain-out*` in the *current thread*. If your program spawns additional threads (e.g. a web server), you can set `s/*explain-out*` for all threads with `(alter-var-root #'s/*explain-out* (constantly expound/printer))`. This won't work (and is not necessary) in CLJS.

[Using `set!` will also not work within an uberjar](https://github.com/bhb/expound/issues/19).

### Error messages for predicates

#### Adding error messages

If a value fails to satisfy a predicate, Expound will print the name of the function (or `<anonymous function>` if the function has no name). To improve the error message, you can use `expound.alpha/def` to add a human-readable error message to the spec.

```clojure
(def email-regex #"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,63}$")

(expound/def :example/email (s/and string? #(re-matches email-regex %)) "should be a valid email address")

(expound/expound :example/email "sally@")

;; -- Spec failed --------------------
;;
;;   "sally@"
;;
;; should be a valid email address
```

If you prefer to use `clojure.spec.alpha/def`, you can still add a message using `expound.alpha/defmsg`:

```clojure
(s/def :ex/name string?)
(expound/defmsg :ex/name "should be a string")
(expound/expound :ex/name :bob)
;; -- Spec failed --------------------
;;
;; :bob
;;
;; should be a string

```

#### Built-in predicates with error messages

Expound provides a default set of type-like predicates with error messages. For example:

```clojure
(expound/expound :expound.specs/pos-int -1)
;; -- Spec failed --------------------
;;
;; -1
;;
;; should be a positive integer
```

You can see the full list of available specs with `expound.specs/public-specs`.


### Configuring the printer

By default, `printer` will omit valid values and replace them with `...`

```clojure
(set! s/*explain-out* expound/printer)
(s/explain :example/place {:city "Denver" :state :CO :country "USA"})

;; -- Spec failed --------------------
;;
;;   {:city ..., :state :CO, :country ...}
;;                      ^^^
;;
;; should satisfy
;;
;;   string?
```

You can configure Expound to show valid values:

```clojure
(set! s/*explain-out* (expound/custom-printer {:show-valid-values? true}))
(s/explain :example/place {:city "Denver" :state :CO :country "USA"})

;; -- Spec failed --------------------
;;
;; {:city "Denver", :state :CO, :country "USA"}
;;                         ^^^
;;
;; should satisfy
;;
;;   string?
```

You can even provide your own function to display the invalid value.

```clojure
;; Your implementation should meet the following spec:
(s/fdef my-value-str
        :args (s/cat
               :spec-name (s/nilable #{:args :fn :ret})
               :form any?
               :path :expound/path
               :value any?)
        :ret string?)
(defn my-value-str [_spec-name form path value]
  (str "In context: " (pr-str form) "\n"
       "Invalid value: " (pr-str value)))

(set! s/*explain-out* (expound/custom-printer {:value-str-fn my-value-str}))
(s/explain :example/place {:city "Denver" :state :CO :country "USA"})

;; -- Spec failed --------------------
;;
;;   In context: {:city "Denver", :state :CO, :country "USA"}
;;   Invalid value: :CO
;;
;; should satisfy
;;
;;   string?
```

### Manual clojure.test/report override

Clojure test allows you to declare a custom multi-method for its `clojure.test/report` function. This is particularly useful in ClojureScript, where a test runner can take care of the boilerplate code:

```clojure
(ns pkg.test-runner
  (:require [cljs.spec.alpha :as s]
            [cljs.test :as test :refer-macros [run-tests]]
            [expound.alpha :as expound]
            ;; require your namespaces here
            [pkg.namespace-test]))

(enable-console-print!)

(set! s/*explain-out* expound/printer)

;; We try to preserve the clojure.test output format
(defmethod test/report [:cljs.test/default :error] [m]
  (test/inc-report-counter! :error)
  (println "\nERROR in" (test/testing-vars-str m))
  (when (seq (:testing-contexts (test/get-current-env)))
    (println (test/testing-contexts-str)))
  (when-let [message (:message m)] (println message))
  (let [actual (:actual m)
        ex-data (ex-data actual)]
    (if (:cljs.spec.alpha/failure ex-data)
      (do (println "expected:" (pr-str (:expected m)))
          (print "  actual:\n")
          (print (.-message actual)))
      (test/print-comparison m))))

;; run tests, (stest/instrument) either here or in the individual test files.
(run-tests 'pkg.namespace-test)
```

The above has been tested in `lumo` so it is self-host ClojureScript compatible.

### Using Orchestra

Use [Orchestra](https://github.com/jeaye/orchestra) with Expound to get human-optimized error messages when checking your `:ret` and `:fn` specs.

```clojure
(require '[orchestra.spec.test :as st])

(s/fdef location
        :args (s/cat :city :example.place/city
                     :state :example.place/state)
        :ret string?)
(defn location [city state]
  ;; incorrect implementation
  nil)

(st/instrument)
(set! s/*explain-out* expound/printer)
(location "Seattle" "WA")

;;ExceptionInfo Call to #'user/location did not conform to spec:
;; form-init3240528896421126128.clj:1
;;
;; -- Spec failed --------------------
;;
;; Return value
;;
;; nil
;;
;; should satisfy
;;
;; string?
;;
;;
;;
;; -------------------------
;; Detected 1 error
```
## Conformers

Expound will not give helpful errors (and in some cases, will throw an exception) if you use conformers to transform values. Although using conformers in this way is fairly common, my understanding is that this is not an [intended use case](https://dev.clojure.org/jira/browse/CLJ-2116).

If you want to use Expound with conformers, you'll need to write a custom printer. See "Configuring the printer" above.

## Related work

- [Inspectable](https://github.com/jpmonettas/inspectable) - Tools to explore specs and spec failures at the REPL
- [Pretty-Spec](https://github.com/jpmonettas/pretty-spec) - Pretty printer for specs
- [Phrase](https://github.com/alexanderkiel/phrase) - Use specs to create error messages for users
- [Pinpointer](https://github.com/athos/Pinpointer) - spec error reporter based on a precise error analysis

## Prior Art

* Error messages in [Elm](http://elm-lang.org/), in particular the [error messages catalog](https://github.com/elm-lang/error-message-catalog)
* Error messages in [Figwheel](https://github.com/bhauman/lein-figwheel), in particular the config error messages generated from [strictly-specking](https://github.com/bhauman/strictly-specking)
* [Clojure Error Message Catalog](https://github.com/yogthos/clojure-error-message-catalog)
* [The Usability of beginner-oriented Clojure error messages](http://wiki.science.ru.nl/tfpie/images/6/6e/TFPIE16-slides-emachkasova.pdf)

## Contributing

Pull requests are welcome, although please open an issue first to discuss the proposed change. I'm also on [clojurians Slack](http://clojurians.net/) as @bbrinck if you have questions.

If you are working on the code, please read the [Development Guide](doc/development.md)

## License

Copyright © 2017-2018 Ben Brinckerhoff

Distributed under the Eclipse Public License version 1.0, just like Clojure.
