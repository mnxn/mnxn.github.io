---
title: "Js_of_ocaml in the VSCode OCaml Platform"
date: 2020-10-13T15:25:33-07:00
draft: true
---



For about two weeks now, the published version of the [VSCode OCaml Platform extension](https://marketplace.visualstudio.com/items?itemName=ocamllabs.ocaml-platform) has had something special about it. 

It's using [Js_of_ocaml](https://ocsigen.org/js_of_ocaml)! This is the result of a month-long effort to switch the extension's OCaml-to-JS compiler from BuckleScript to Js_of_ocaml.



## The OCaml Platform

The VSCode OCaml Platform extension is part of the larger [OCaml Platform](https://www.youtube.com/watch?v=E8T_4zqWmq8). 

The VSCode extension interacts directly with [OCaml-LSP](https://github.com/ocaml/ocaml-lsp/), an implementation of the [Language Server Protocol](https://microsoft.github.io/language-server-protocol) for OCaml editor support. The OCaml-LSP language server provides editor features like code completion, go to definition, formatting, and error highlighting. 

OCaml-LSP can be used from any editor that supports the LSP, but the VSCode extension provides additional features: managing different package manager sandboxes; syntax highlighting of many OCaml-related filetypes; and integration with VSCode tasks, snippets, and indentation rules.

Both the language server and VSCode extension are continuously tested to ensure compatibility for opam and esy on macOS, Linux, and [even Windows](https://github.com/ocamllabs/vscode-ocaml-platform#windows).



## BuckleScript vs. Js_of_ocaml

BuckleScript and Js_of_ocaml are technologies that accomplish a similar goal: compiling OCaml to Javascript code. However, there are a few difference that made it worthwhile to switch to Js_of_ocaml. 

The ways in which BuckleScript and Js_of_ocaml approach compiling to JS are fundamentally different. BuckleScript compiles from  an early intermediate representation of the OCaml compiler to generate small JS files for each OCaml module; this is effective but   fixes the OCaml language to a certain version (4.06.1 at the time of writing). Js_of_ocaml takes a different approach and generates JS from OCaml bytecode, which is more stable across different versions of OCaml. Using Js_of_ocaml allows the VSCode extension to be built with a recent version of OCaml (4.11.1 right now). 

BuckleScript has undergone a rebranding and it is now called [ReScript](rescript-lang.org/) with the addition of a new syntax. I'm personally indifferent to the new syntax, but [the removal of the OCaml documentation](https://discuss.ocaml.org/t/where-do-i-look-for-docs-now-that-bucklescript-is-gone/6283) did not sit well with me. As I revisit the site today, OCaml and ReasonML have returned in the old v8.0.0 documentation as "Older Syntax" and "Old Syntax", respectively. It does seem that OCaml and ReasonML will be technically supported for now (forever?), but the project's distancing from those languages does not inspire confidence.

Js_of_ocaml, on the other hand, is deeply integrated with the OCaml language and ecosystem. This is great for existing OCaml developers because it means complex projects that compile to JavaScript can be built with the excellent [dune](https://github.com/ocaml/dune) build system with access to most of the same opam packages as native OCaml projects. 



## gen_js_api

For a VSCode extension, there are a lot of bindings that have to be made to interact with the VSCode extension api. For the BuckleScript version of the extension, we used the built-in syntax for bindings. 

For example, to bind to [`vscode.window.createOutputChannel`](https://code.visualstudio.com/api/references/vscode-api#window.createOutputChannel) in BuckleScript:

```ocaml
external createOutputChannel : name:string -> OutputChannel.t
  = "createOutputChannel"
  [@@bs.module "vscode"] [@@bs.scope "window"]
```

This expresses that `createOutputChannel` is a function from the `window` namespace of the `vscode` node module. The `bs.module` annotation automatically inserts the `require` for the `vscode` module in the generated JavaScript.

There are a few ways to express the same things in Js_of_ocaml. The first is to use the provided functions in the `Js_of_ocaml.Js` module to manually preform conversions between OCaml and JS types:

```ocaml
let createOutputChannel ~name =
  Js.Unsafe.global##.vscode##.window##createOutputChannel [| Js.string name |]
```

Notice that certain OCaml types (like strings) have to be converted into their JS representation. Doing these conversions manually may work for small libraries, but it would be impractical to do that for every binding and type in the VSCode API. 

For that reason, [gen_js_api](https://github.com/LexiFi/gen_js_api) is a great alternative. The same binding with gen_js_api looks like this:

```ocaml
val createOutputChannel : name:string -> OutputChannel.t
  [@@js.global "vscode.window.createOutputChannel"]
```

What gen_js_api will do is generate code that automatically calls `Ojs.string_to_js` for the parameter and `OutputChannel.t_of_js` for the return value. An OCaml value can be converted a JS value if it is a ["JS-able"](https://github.com/LexiFi/gen_js_api/blob/master/TYPES.md) type, or if the appropriate `of_js`/`to_js` functions already exist. 

It is important to note that unlike BuckleScript, it is actually doing a conversion between values. If a function binding is written that returns an OCaml record, modifying a field of that record only modifies the value returned from the `to_js` function; the original JS value is untouched. This is different from BuckleScript, where an OCaml type directly corresponding to its JS data representation.

To avoid this, it is possible to keep values as an abstract `Ojs.t` type, which are the unconverted JS values. Accessing and setting fields can be done with function annotated with [`[@@js.set]` or `[@@js.get]`](https://github.com/LexiFi/gen_js_api/blob/master/VALUES.md). This method was used for the entirety of the extension's VSCode bindings, which can be found [here](https://github.com/ocamllabs/vscode-ocaml-platform/tree/master/vscode).



## Node Modules

As mentioned previously, BuckleScript provides a way to reference Node modules with `[@@bs.module]`. As far as I know, there is no simple equivalent provided by Js_of_ocaml or gen_js_api. Fortunately there is a simple workaround with a single JS stub file:

```js
joo_global_object.vscode = require("vscode");
```

In this JavaScript file, `joo_global_object` refers to the same value as `Js.Unsafe.global`. Setting the `vscode` field this way allows it to be referenced by the gen_js_api functions globally.

For this JavaScript file to be used, the dune configuration for the library must be updated with:

```text
(js_of_ocaml (javascript_files vscode_stub.js))
```



## JSON



## Promises

- Async testing libraries?



## Sys.unix



## Closing

