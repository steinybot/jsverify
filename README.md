# JSVerify

<img src="https://raw.githubusercontent.com/jsverify/jsverify/master/jsverify-300.png" align="right" height="100" />

> Property based checking. Like QuickCheck.

[![Build Status](https://secure.travis-ci.org/jsverify/jsverify.svg?branch=master)](http://travis-ci.org/jsverify/jsverify)
[![NPM version](https://badge.fury.io/js/jsverify.svg)](http://badge.fury.io/js/jsverify)
[![Dependency Status](https://david-dm.org/jsverify/jsverify.svg)](https://david-dm.org/jsverify/jsverify)
[![devDependency Status](https://david-dm.org/jsverify/jsverify/dev-status.svg)](https://david-dm.org/jsverify/jsverify#info=devDependencies)
[![Code Climate](https://img.shields.io/codeclimate/github/jsverify/jsverify.svg)](https://codeclimate.com/github/jsverify/jsverify)

## Getting Started

Install the module with: `npm install jsverify`

## Synopsis

```js
var jsc = require("jsverify");

// forall (f : bool -> bool) (b : bool), f (f (f b)) = f(b).
var boolFnAppliedThrice =
  jsc.forall("bool -> bool", "bool", function (f, b) {
    return f(f(f(b))) === f(b);
  });

jsc.assert(boolFnAppliedThrice);
// OK, passed 100 tests
```


## Documentation

### Usage with [mocha](http://visionmedia.github.io/mocha/)

Using jsverify with mocha is easy, just define the properties and use `jsverify.assert`.

Starting from version 0.4.3 you can write your specs without any boilerplate:

```js
describe("sort", function () {
  jsc.property("idempotent", "array nat", function (arr) {
    return _.isEqual(sort(sort(arr)), sort(arr));
  });
});
```

You can also provide `--jsverifyRngState state` command line argument, to run tests with particular random generator state.

```
$ mocha examples/nat.js

1) natural numbers are less than 90:
 Error: Failed after 49 tests and 1 shrinks. rngState: 074e9b5f037a8c21d6; Counterexample: 90;

$ mocha examples/nat.js --grep 'are less than' --jsverifyRngState 074e9b5f037a8c21d6

1) natural numbers are less than 90:
   Error: Failed after 1 tests and 1 shrinks. rngState: 074e9b5f037a8c21d6; Counterexample: 90;
```

Errorneous case is found with first try.

### Usage with [jasmine](http://pivotal.github.io/jasmine/)

Check [jasmineHelpers.js](helpers/jasmineHelpers.js) and [jasmineHelpers2.js](helpers/jasmineHelpers2.js) for jasmine 1.3 and 2.0 respectively.

## API

> _Testing shows the presence, not the absence of bugs._
>
> Edsger W. Dijkstra

To show that propositions hold, we need to construct proofs.
There are two extremes: proof by example (unit tests) and formal (machine-checked) proof.
Property-based testing is somewhere in between.
We formulate propositions, invariants or other properties we believe to hold, but
only test it to hold for numerous (randomly generated) values.

Types and function signatures are written in [Coq](http://coq.inria.fr/)/[Haskell](http://www.haskell.org/haskellwiki/Haskell) influented style:
C# -style `List<T> filter(List<T> v, Func<T, bool> predicate)` is represented by
`filter (v : array T) (predicate : T -> bool) : array T` in our style.

`jsverify` can operate with both synchronous and asynchronous-promise properties.
Generally every property can be wrapped inside [functor](http://learnyouahaskell.com/functors-applicative-functors-and-monoids),
for now in either identity or promise functor, for synchronous and promise properties respectively.


### Properties


- `forall(arbs: arbitrary a ..., userenv: (map arbitrary)?, prop : a -> property): property`

    Property constructor


- `check (prop: property, opts: checkoptions?): result`

    Run random checks for given `prop`. If `prop` is promise based, result is also wrapped in promise.

    Options:
    - `opts.tests` - test count to run, default 100
    - `opts.size`  - maximum size of generated values, default 5

    - `opts.quiet` - do not `console.log`
    - `opts.rngState` - state string for the rng


- `assert(prop: property, opts: checkoptions?) : void`

    Same as `check`, but throw exception if property doesn't hold.


- `property(name: string, ...)`

   Assuming there is globally defined `it`, the same as:

   ```js
   it(name, function () {
     jsc.assert(jsc.forall(...));
   }
   ```

   You can use `property` to write facts too:
   ```js
   jsc.property("+0 === -0", function () {
     return +0 === -0;
   });
   ```


- `sampler(arb: arbitrary a, genSize: nat = 10): (sampleSize: nat?) -> a`

    Create a sampler for a given arbitrary with an optional size. Handy when used in
    a REPL:
    ```
    > jsc = require('jsverify') // or require('./lib/jsverify') w/in the project
    ...
    > jsonSampler = jsc.sampler(jsc.json, 4)
    [Function]
    > jsonSampler()
    0.08467432763427496
    > jsonSampler()
    [ [ [] ] ]
    > jsonSampler()
    ''
    > sampledJson(2)
    [-0.4199344692751765, false]
    ```


### Types

- `generator a` is a function `(size: nat) -> a`.
- `show` is a function `a -> string`.
- `shrink` is a function `a -> [a]`, returning *smaller* values.
- `arbitrary a` is a triple of generator, shrink and show functions.
    - `{ generator: nat -> a, shrink : a -> array a, show: a -> string }`


### DSL for input parameters

There is a small DSL to help with `forall`. For example the two definitions below are equivalent:
```js
var bool_fn_applied_thrice = jsc.forall("bool -> bool", "bool", check);
var bool_fn_applied_thrice = jsc.forall(jsc.fn(jsc.bool()), jsc.bool(), check);
```

The DSL is based on a subset of language recognized by [typify-parser](https://github.com/phadej/typify-parser):
- *identifiers* are fetched from the predefined environment.
- *applications* are applied as one could expect: `"array bool"` is evaluated to `jsc.array(jsc.bool)`.
- *functions* are supported: `"bool -> bool"` is evaluated to `jsc.fn(jsc.bool())`.
- *square brackets* are treated as a shorthand for the array type: `"[nat]"` is evaluated to `jsc.array(jsc.nat)`.



### Primitive arbitraries


- `integer: arbitrary integer`
- `integer(maxsize: nat): arbitrary integer`
- `integer(minsize: integer, maxsize: integer): arbitrary integer`

    Integers, ℤ


- `nat: arbitrary nat`
- `nat(maxsize: nat): arbitrary nat`

    Natural numbers, ℕ (0, 1, 2...)


- `number: arbitrary number`
- `number(maxsize: number): arbitrary number`
- `number(min: number, max: number): arbitrary number`

    JavaScript numbers, "doubles", ℝ. `NaN` and `Infinity` are not included.


- `uint8: arbitrary nat`
- `uint16: arbitrary nat`
- `uint32: arbitrary nat`


- `int8: arbitrary integer`
- `int16: arbitrary integer`
- `int32: arbitrary integer`


- `bool: arbitrary bool`

    Booleans, `true` or `false`.


- `datetime: arbitrary datetime`

    Random datetime


- `elements(args: array a): arbitrary a`

    Random element of `args` array.


- `char: arbitrary char`

    Single character


- `asciichar: arbitrary char`

    Single ascii character (0x20-0x7e inclusive, no DEL)


- `string: arbitrary string`


- `notEmptyString: arbitrary string`

    Generates strings which are not empty.


- `asciistring: arbitrary string`


- `json: arbitrary json`

     JavaScript Objects: boolean, number, string, array of `json` values or object with `json` values.

- `value: arbitrary json`


- `falsy: arbitrary *`

    Generates falsy values: `false`, `null`, `undefined`, `""`, `0`, and `NaN`.


- `constant(x: a): arbitrary a`

    Returns an unshrinkable arbitrary that yields the given object.



### Arbitrary combinators


- `nonshrink(arb: arbitrary a): arbitrary a`

    Non shrinkable version of arbitrary `arb`.


- `array(arb: arbitrary a): arbitrary (array a)`


- `nearray(arb: arbitrary a): arbitrary (array a)`


- `pair(arbA: arbitrary a, arbB : arbitrary b): arbitrary (pair a b)`

    If not specified `a` and `b` are equal to `value()`.


- `tuple(arbs: (arbitrary a, arbitrary b...)): arbitrary (a, b...)`


- `map(arb: arbitrary a): arbitrary (map a)`

    Generates a JavaScript object with properties of type `A`.


- `oneof(gs : array (arbitrary a)...) : arbitrary a`

    Randomly uses one of the given arbitraries.


- `record(spec: { key: arbitrary a... }): arbitrary { key: a... }`

    Generates a javascript object with given record spec.



- `fn(gen: generator a): generator (b -> a)`
- `fun(gen: generator a): generator (b -> a)`
    Unary functions.



### Generator functions


- `generator.constant(x: a): gen a`


- `generator.pair(genA: gen a, genB: gen b, size: nat): gen (a, b)`


- `generator.tuple(gens: (gen a, gen b...), size: nat): gen (a, b...)`


- `generator.array(gen: gen a, size: nat): gen (array a)`


- `generator.nearray(gen: Gen a, size: nat): gen (array a)`


- `generator.char: gen char`


- `generator.string(size: nat): gen string`


- `generator.nestring(size: nat): gen string`


- `generator.asciichar: gen char`


- `generator.asciistring(size: nat): gen string`


- `generator.map(gen: gen a, size: nat): gen (map a)`


- `generator.oneof(gen: list (gen a), size: nat): gen a`


- `generator.combine(gen: gen a..., f: a... -> b): gen b`


- `generator.recursive(genZ: gen a, genS: gen a -> gen a): gen a`


- `generator.json: gen json`



### Shrink functions


- `shrink.noop(x: a): array a`


- `shrink.pair(shrA: a -> array a, shrB: b -> array, x: (a, b)): array (a, b)`


- `shrink.tuple(shrinks: (a -> array a, b -> array b...), x: (a, b...)): array (a, b...)`


- `shrink.array(shrink: a -> array a, x: array a): array (array a)`


- `shrink.nearray(shrink: a -> nearray a, x:  nearray a): array (nearray a)`


- `shrink.record(shrinks: { key: a -> string... }, x: { key: a... }): array { key: a... }`



### Show functions


- `show.def(x : a): string`


- `show.pair(showA: a -> string, showB: b -> string, x: (a, b)): string`


- `show.tuple(shrinks: (a -> string, b -> string...), x: (a, b...)): string`


- `show.array(shrink: a -> string, x: array a): string`



### Random functions


- `random(min: int, max: int): int`

    Returns random int from `[min, max]` range inclusively.

    ```js
    getRandomInt(2, 3) // either 2 or 3
    ```


- `random.number(min: number, max: number): number`

    Returns random number from `[min, max)` range.



### Utility functions

Utility functions are exposed (and documented) only to make contributions to jsverify more easy.
The changes here don't follow semver, i.e. there might be backward-incompatible changes even in patch releases.

Use [underscore.js](http://underscorejs.org/), [lodash](https://lodash.com/), [ramda](http://ramda.github.io/ramdocs/docs/), [lazy.js](http://danieltao.com/lazy.js/) or some other utility belt.


- `utils.isEqual(x: json, y: json): bool`

    Equality test for `json` objects.


- `utils.force(x: a | () -> a) : a`

    Evaluate `x` as nullary function, if it is one.


- `utils.merge(x: obj, y: obj): obj`

  Merge two objects, a bit like `_.extend({}, x, y)`.



## Contributing

- `README.md` is generated from the source with [ljs](https://github.com/phadej/ljs)
- `jsverify.standalone.js` is also generated by the build process
- Before creating a pull request run `make test`, yet travis will do it for you.

## Release History

- **0.5.0-beta.2** &mdash; *2014-12-21* &mdash; Beta 2!
    - Pair &amp; tuple related code cleanup
    - Update `CONTRIBUTING.md`
    - Small documentation type fixes
    - Bless `jsc.elements` shrink
- **0.5.0-beta.1** &mdash; *2014-12-20* &mdash; Beta!
    - `bless` don't close over (uses `this`)
    - Cleanup generator module
    - Other code cleanup here and there
- **0.4.6** &mdash; *2014-11-30*  &mdash; better shrinks &amp; recursive
    - Implemented shrinks: [#51](https://github.com/jsverify/jsverify/issues/51)
    - `jsc.generator.recursive`: [#37](https://github.com/jsverify/jsverify/issues/37)
    - array, nearray &amp; map generators return a bit smaller results (*log2* of size)
- **0.4.5** &mdash; *2014-11-22*  &mdash; stuff
    - `generator.combine` &amp; `.flatmap`
    - `nat`, `integer`, `number` &amp; and `string` act as objects too
- **0.4.4** &mdash; *2014-11-22*  &mdash; new generators
    - New generators: `nearray`, `nestring`
    - `generator.constant`
    - zero-ary `jsc.property` (it ∘ assert)
    - `jsc.sampler`
- **0.4.3** &mdash; *2014-11-08*  &mdash; jsc.property
    - Now you can write your bdd specs without any boilerplate
    - support for nat-litearls in dsl [#36](https://github.com/jsverify/jsverify/issues/36)
        ```js
        describe("Math.abs", function () {
          jsc.property("result is non-negative", "integer 100", function (x) {
            return Math.abs(x) >= 0;
          });
        });
        ```
    - Falsy generator [#42](https://github.com/jsverify/jsverify/issues/42)
- **0.4.2** &mdash; *2014-11-03*  &mdash; User environments for DSL
    - User environments for DSL
    - Generator prototype `map`, and shrink prototype `isomap`
    - JSON generator works with larger sizes
- **0.4.1** Move to own organization in GitHub
- **0.4.0**  &mdash; *2014-10-27*  &mdash; typify-dsl &amp; more arbitraries.
    Changes from **0.3.6**:
    - DSL for `forall` and `suchthat`
    - new primitive arbitraries
    - `oneof` behaves as in QuickCheck (BREAKING CHANGE)
    - `elements` is new name of old `oneof`
    - Other smaller stuff under the hood
- **0.4.0**-beta.4 generator.oneof
- **0.4.0**-beta.3 Expose shrink and show modules
- **0.4.0**-beta.2 Move everything around
    - Better looking README.md!
- **0.4.0**-beta.1 Beta!
    - Dev Dependencies update
- **0.4.0**-alpha8 oneof &amp; record -dsl support
    - also `jsc.compile`
    - record is shrinkable!
- **0.4.0**-alpha7 oneof &amp; record
    - *oneof* and *record* generator combinators ([@fson](https://github.com/fson))
    - Fixed uint\* generators
    - Default test size increased to 10
    - Numeric generators with size specified are independent of test size ([#20](https://github.com/phadej/jsverify/issues/20))
- **0.4.0**-alpha6 more primitives
    - int8, int16, int32, uint8, uint16, uint32
    - char, asciichar and asciistring
    - value &rarr; json
    - use eslint
- **0.4.0**-alpha5 move david to be devDependency
- **0.4.0**-alpha4 more typify
    - `suchchat` supports typify dsl
    - `oneof` &rarr; `elements` to be in line with QuickCheck
    - Added versions of examples using typify dsl
- **0.4.0**-alpha3 David, npm-freeze and jscs
- **0.4.0**-alpha2 Fix typo in readme
- **0.4.0**-alpha1 typify
   - DSL for `forall`
       ```js
       var bool_fn_applied_thrice = jsc.forall("bool -> bool", "bool", check);
       ```

   - generator arguments, which are functions are evaluated. One can now write:
       ```js
       jsc.forall(jsc.nat, check) // previously had to be jsc.nat()
       ```

- **0.3.6** map generator
- **0.3.5** Fix forgotten rngState in console output
- **0.3.4** Dependencies update
- **0.3.3** Dependencies update
- **0.3.2** `fun` &rarr; `fn`
- **0.3.1** Documentation typo fixes
- **0.3.0** Major changes
    - random generate state handling
    - `--jsverifyRngState` parameter value used when run on node
    - karma tests
    - use make
    - dependencies update
- **0.2.0** Use browserify
- **0.1.4** Mocha test suite
    - major cleanup
- **0.1.3** gen.show and exception catching
- **0.1.2** Added jsc.assert
- **0.1.1** Use grunt-literate
- **0.1.0** Usable library
- **0.0.2** Documented preview
- **0.0.1** Initial preview

## Related work

### JavaScript

- [JSCheck](http://www.jscheck.org/)
- [claire](https://npmjs.org/package/claire)
- [gent](https://npmjs.org/package/gent)
- [fatcheck](https://npmjs.org/package/fatcheck)
- [quickcheck](https://npmjs.org/package/quickcheck)
- [qc.js](https://bitbucket.org/darrint/qc.js/)
- [quick\_check](https://www.npmjs.org/package/quick_check)
- [gencheck](https://github.com/graue/gentest)
- [node-quickcheck](https://github.com/mcandre/node-quickcheck)

### Others

- [Wikipedia - QuickCheck](http://en.wikipedia.org/wiki/QuickCheck)
- [Haskell - QuickCheck](http://hackage.haskell.org/package/QuickCheck) [Introduction](http://www.haskell.org/haskellwiki/Introduction_to_QuickCheck1)
- [Erlang - QuviQ](http://www.quviq.com/index.html)
- [Erlang - triq](https://github.com/krestenkrab/triq)
- [Scala - ScalaCheck](https://github.com/rickynils/scalacheck)
- [Clojure - test.check](https://github.com/clojure/test.check)

The MIT License (MIT)

Copyright (c) 2013, 2014 Oleg Grenrus

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
