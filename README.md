# [OCamlScript](http://bloomberg.github.io/ocamlscript/)

## Introduction
OCamlScript is a JavaScript backend for [the OCaml compiler](https://ocaml.org/). Users of OCamlScript can write type-safe, high performance OCaml code, and deploy the generated JavaScript in any platform with a JavaScript execution engine.

Each OCaml module is mapped to a corresponding JavaScript module, and names are preserved so that:

1. The stacktrace is preserved, the generated code is debuggable with or without sourcemap.
2. Modules generated by OCamlScript can be used directly from other JavaScript code. For example, you can call the `List.length` function from the OCaml standard library `List` module from JavaScript.

### A simple example

``` ocaml
let sum n =
    let v  = ref 0 in
    for i = 0 to n do
       v := !v + i
    done;
    !v
```

OCamlScript generates code similar to this:

``` js
function sum(n) {
  var v = 0;
  for(var i = 0; i<= n; ++i){
    v += i;
  }
  return v;
}
```

As you can see, there is no name mangling in the generated code, so if this module is called `M`,
`M.sum()` is directly callable from other JavaScript code.

You can learn more by exploring the online [in-browser compiler](https://bloomberg.github.io/ocamlscript/js-demo/).

  
## Disclaimer

This project has been released to exchange ideas and collect feedback from the OCaml and JavaScript communities.
It is in an *very early* stage and not production ready for your own projects *yet*.


## Build

Note that you have to clone this project with `--recursive` option, as the core OCaml compiler is brought into your clone as a Git `submodule`.

### Linux and Mac OSX


1. Apply the patch to OCaml compiler and build

  Please ignore the warnings generated by git apply
  
  ```sh
  cd ocaml
  git apply ../js.diff
  ./configure -prefix `pwd`
  make world.opt
  make install
  ```

2. Build OCamlScript Compiler

  Assume that you have `ocamlopt.opt` in the `PATH`
  ```sh
  cd ../jscomp
  ocamlopt.opt -I +compiler-libs -I bin -c bin/compiler.mli bin/compiler.ml
  ocamlopt.opt -g -linkall -o bin/ocamlscript -I +compiler-libs ocamlcommon.cmxa ocamlbytecomp.cmxa  bin/compiler.cmx main.cmx
  ```
  Now you have a binary called `ocamlscript` under `jscomp/bin` directory,
  put it in your `PATH`.
  
3. Build the runtime with `ocamlscript`

  ```sh
  cd runtime
  make all
  ```

4. Build the standard library with `ocamlscript`

  ```sh
  cd ../stdlib
  make all 
  ```

5. Test 

  Create a file called `hello.ml`:

  ```sh
  mkdir tmp  # create tmp directory inside the stdlib
  cd tmp 
  echo 'print_endline "hello world";;' >hello.ml
  ```
  
  Then compile it with `ocamlscript`
  ```sh
  OCAML_RAW_JS=1 ocamlscript -I . -I ../ -c hello.ml
  ```
  
  It should generate a file called `hello.js`, which can be executed with any JavaScript engine. In this example, we use Node.js

  ```sh
  node hello.js
  ```
  
  If everything goes well, you will see `hello world` on your screen.

## Windows support

We plan to provide a Windows installer in the near future.

# Licensing 

The [OCaml](./ocaml) directory is the official OCaml compiler (version 4.02.3). Refer to its copyright and license notices for information about its licensing.

The `ocamlscript` backend relies on a patch [(js.diff)](./js.diff) to the OCaml compiler.

This project reused and adapted parts of [js_of_ocaml](https://github.com/ocsigen/js_of_ocaml):
* Some small printing utilities in [pretty printer](./jscomp/js_dump.ml).
* Part of the [Javascript runtime](./jscomp/runtime) support

It adapted two modules [Lam_pass_exits](jscomp/lam_pass_exits.ml) and
[Lam_pass_lets_dce](jscomp/lam_pass_lets_dce.ml) from OCaml's
[Simplif](ocaml/bytecomp/simplif.ml) module, the main
reasons are those optimizations are not optimal for Javascript
backend.

[Js_main](jscomp/js_main.ml) is adapted from [driver/main](ocaml/driver/main.ml), it is not actually
used, since currently we make this JS backend as a plugin instead, but
it shows that it is easy to assemble a whole compler using OCaml
compiler libraries and based upon that we can add more compilation flags for
JS backend.

[stdlib](jscomp/stdlib) is copied from ocaml's [stdlib](ocaml/stdlib) to have it compiled with
the new JS compiler.

Since our work is derivative work of [js_of_ocaml](http://ocsigen.org/js_of_ocaml/), the license of the OCamlScript components is GPLv2, the same as js_of_ocaml.

## Design goals

1. Readability 
 1.   No name mangling.
 2.   Support JavaScript module system.
 3.   Integrate with existing JavaScript ecosystem, for example,
      [npm](https://www.npmjs.com/), [webpack](https://github.com/webpack).
 4.   Straight-forward FFI, generate tds file to target [Typescript](http://www.typescriptlang.org/) for better tooling.
 
2. Separate and *extremely fast* compilation.

3. Better performance than hand-written Javascript:
   Thanks to the solid type system in OCaml it is possible to optimize the generated JavaScript code.

4. Smaller code than hand written JavaScript, aggressive dead code elimination.

5. Support Node.js, web browsers, and various Javascript target platforms.

6. Compatible with OCaml semantics modulo c-bindings and Obj, Marshal modules.

## More examples

### A naive benchmark comparing to the Immutable map from Facebook

Below is a *contrived* example to demonstrate our motivation,
it tries to insert 1,000,000 keys into an immutable map, then query it.

```Ocaml
module IntMap = Map.Make(struct
  type t = int
  let compare (x : int) y = compare x y
  end)

let test () =
  let m = ref IntMap.empty in
  let count = 1000000 in
  for i = 0 to count do
    m := IntMap.add i i !m
  done;
  for i = 0 to count  do
    ignore (IntMap.find i !m )
  done

let () = test()

```

The code generated by OCamlScript is a drop-in replacement for the Facebook [immutable](http://facebook.github.io/immutable-js/) library.


``` js
'use strict';
var Immutable = require('immutable');
var Map = Immutable.Map;
var m = new Map();
var test  = function(){
    var count  = 1000000
    for(var i = 0; i < count; ++i){
        m = m.set(i, i );
    }
    for(var j = 0; j < count ; ++j){
        m.get(j)
    }
}

test ()
```

Runtime performance:
- OCamlScript Immutable Map: 1186ms
- Facebook Immutable Map: 3415ms

Code Size:
- OCamlScript (Prod mode): 899 Bytes
- Facebook Immutable : 55.3K Bytes

## Status

While most of the OCaml language is covered, because this project is still young there is plenty of work left to be done.

Some known issues are listed as below:

1. Language features:

   Recursive modules, have not looked into it yet.
   
   Better Currying support. Currently, we have an inference engine for
   function currying and we do cross module inference, however, there
   are some more challenging cases, for example, high order functions,
   it can be resolved by either aggressive inlining or falling back to a
   slow path using `Function.prototype.length`. We prepared the
   runtime support in [module curry](jscomp/runtime/curry.ml), will support it in the near
   future.

   Int32 operations, currently, Int use Float operations, this should
   be fixed in the near future.
   
2. Standard libraries distributed with OCaml:

   IO support, we have very limited support for
   `Pervasives.print_endline` and `Pervasives.prerr_endline`, it's
   non-trivial to preserve the same semantics of IO between OCaml and
   Node.js, one solution is to functorize all IO operations. Functors
   are then inlined so there will no be performance cost or code size
   penalty.

   Bigarray, Unix, Num, Int64

3. String is immutable, user is expected to compile with flags `-safe-string` for all modules:

   Note that this flag should be applied to all your modules.

## Question, Comments and Feedback

If you have questions, comments, suggestions for improvement or any other inquiries
regarding this project, feel free to open an issue in the issue tracker.
