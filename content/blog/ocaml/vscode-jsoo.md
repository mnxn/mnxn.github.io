---
title: "Js_of_ocaml in the VSCode OCaml Platform"
date: 2020-10-20T23:51:37-07:00
description: "Lessons in porting the VSCode OCaml Platform extension from BuckleScript to Js_of_ocaml."
---

For about two weeks now, the published version of the [VSCode OCaml Platform extension](https://marketplace.visualstudio.com/items?itemName=ocamllabs.ocaml-platform) has had something special about it.

It is using [Js_of_ocaml](https://ocsigen.org/js_of_ocaml)! This is the result of a month-long effort to switch the extension's OCaml-to-JS compiler from BuckleScript to Js_of_ocaml.

In this post, I will describe the extension, explain the reasoning for switching to Js_of_ocaml, and go over some of the things I learned through the porting experience.



## The OCaml Platform

The VSCode OCaml Platform extension is part of the larger [OCaml Platform](https://www.youtube.com/watch?v=E8T_4zqWmq8); it interacts directly with [OCaml-LSP](https://github.com/ocaml/ocaml-lsp/), an implementation of the [Language Server Protocol](https://microsoft.github.io/language-server-protocol) for OCaml editor support. The OCaml-LSP language server provides editor features like code completion, go to definition, formatting with [ocamlformat](https://github.com/ocaml-ppx/ocamlformat), and error highlighting.

OCaml-LSP can be used from any editor that supports the protocol, but the VSCode extension provides additional features: managing different package manager sandboxes; syntax highlighting of many OCaml-related filetypes; and integration with VSCode tasks, snippets, and indentation rules.

Both the language server and VSCode extension are continuously tested to ensure compatibility for opam and esy on macOS, Linux, and [even Windows](https://github.com/ocamllabs/vscode-ocaml-platform#windows).

Making OCaml more accessible is a goal for these projects. Providing support for a popular cross-platform code editor and the widely supported language server protocol helps achieve that goal.



## BuckleScript vs. Js_of_ocaml

BuckleScript and Js_of_ocaml are technologies that accomplish a similar goal: compiling OCaml to Javascript code. However, there are a few differences that made it worthwhile to switch to Js_of_ocaml.

The ways in which BuckleScript and Js_of_ocaml approach compiling to JS are notably different. BuckleScript compiles from an early intermediate representation of the OCaml compiler to generate small JS files for each OCaml module; this is effective but fixes the OCaml language to a certain version (4.06.1 at the time of writing). Js_of_ocaml takes a different approach and generates JS from OCaml bytecode, which is more stable across different versions of OCaml but might provide less information. Using Js_of_ocaml allows the VSCode extension to be built with a recent version of OCaml (4.11.1 right now).

BuckleScript has undergone a rebranding and it is now called [ReScript](https://rescript-lang.org/) with the addition of a new syntax. At one point, OCaml documentation was [removed from the website](https://discuss.ocaml.org/t/where-do-i-look-for-docs-now-that-bucklescript-is-gone/6283). As I revisit the site today, OCaml documentation has returned in the old v8.0.0 documentation as "Older Syntax". It does seem that OCaml and ReasonML will be technically supported for now (forever?), but the project feels more distant from OCaml than it did as BuckleScript.

Js_of_ocaml, on the other hand, is deeply integrated with the OCaml language and ecosystem. This integration is great for existing OCaml developers because it means complex JS projects can be built with the excellent [dune](https://github.com/ocaml/dune) build system with access to most of the same opam packages as native projects.

For a more in-depth comparison of the two technologies, I recommend reading [@jchavarri](https://github.com/jchavarri)'s [post about the topic](https://www.javierchavarri.com/js_of_ocaml-and-bucklescript/).



## gen_js_api

For a VSCode extension, there are many bindings that have to be created for interaction with the VSCode extension API. For the BuckleScript version of the extension, we used the built-in syntax for bindings.

For example, to bind to [`vscode.window.createOutputChannel`](https://code.visualstudio.com/api/references/vscode-api#window.createOutputChannel) in BuckleScript:

```ocaml
external createOutputChannel : name:string -> OutputChannel.t
  = "createOutputChannel"
  [@@bs.module "vscode"] [@@bs.scope "window"]
```

This expresses that `createOutputChannel` is a function from the `window` namespace of the `vscode` node module. The `bs.module` annotation automatically inserts the `require` for the `vscode` module in the generated JavaScript.

There are a few ways to express the same things in Js_of_ocaml. The first is to use the provided functions in the `Js_of_ocaml.Js` module to manually convert between OCaml and JS types:

```ocaml
let createOutputChannel ~name =
  Js.Unsafe.global##.vscode##.window##createOutputChannel [| Js.string name |]
```

Notice that certain OCaml types (like strings) have to be converted into their JS representation. Doing these conversions manually may work for small libraries, but it would be impractical to do that for every binding and type in the expansive VSCode API.

For that reason, [gen_js_api](https://github.com/LexiFi/gen_js_api) is a great alternative. The same binding with gen_js_api looks like this:

```ocaml
val createOutputChannel : name:string -> OutputChannel.t
  [@@js.global "vscode.window.createOutputChannel"]
```

What gen_js_api will do is generate code that automatically calls `Ojs.string_to_js` for the parameter and `OutputChannel.t_of_js` for the return value. An OCaml value can be converted a JS value if it is a ["JS-able"](https://github.com/LexiFi/gen_js_api/blob/master/TYPES.md) type, or if the appropriate `of_js`/`to_js` functions exist.

It is important to note that unlike BuckleScript, gen_js_api is actually doing a conversion between values. If a function binding is written that returns an OCaml record, modifying a field of that record only modifies the record itself; the original JS value is untouched. This is different from BuckleScript, where an OCaml type directly corresponding to its JS data representation.

To avoid this, it is possible to keep values as an abstract `Ojs.t` type, which are the unconverted JS values. Accessing and setting fields can be done with a function annotated with [`[@@js.set]` or `[@@js.get]`](https://github.com/LexiFi/gen_js_api/blob/master/VALUES.md). This method was used for the entirety of the extension's VSCode bindings, which can be found [here](https://github.com/ocamllabs/vscode-ocaml-platform/tree/master/vscode).



## Node Modules

As mentioned previously, BuckleScript provides a way to reference Node modules with `[@@bs.module]`. As far as I know, there is no simple equivalent provided by Js_of_ocaml or gen_js_api at the moment. Fortunately, there is a simple workaround with a single JS stub file:

```js
joo_global_object.vscode = require("vscode");
```

In this JavaScript file, `joo_global_object` refers to the same value as `Js.Unsafe.global`. Setting the `vscode` field this way allows it to be referenced by the gen_js_api functions globally.

For this JavaScript file to be used, the library's dune configuration must be updated with:

```text
(js_of_ocaml (javascript_files vscode_stub.js))
```

Afterward, bindings can be created that reference `vscode` and its namespaces or values.

There is some [ongoing work](https://github.com/LexiFi/gen_js_api/pull/119) in gen_js_api that may improve the interaction between node modules and scopes.



## JSON

JSON is a staple of many browser and node.js projects; the VSCode extension is no different. In the extension, JSON is used to (de)serialize user settings and interact with the language server using JSON-RPC.

When the extension was built with BuckleScript, it used [@glennsl](https://github.com/glennsl)'s [bs-json](https://github.com/glennsl/bs-json) library for composable encoding and decoding functions. bs-json wasn't available for Js_of_ocaml, so I decided to reimplement it for Js_of_ocaml as [jsonoo](https://github.com/mnxn/jsonoo) ([opam](https://opam.ocaml.org/packages/jsonoo/)). The documentation for jsonoo is available [here](https://github.com/mnxn/jsonoo).

The main idea is that there are `decoder` and `encoder` types which are `Jsonoo.t -> 'a` and `'a -> Jsonoo.t` function types, respectively. The functions provided by jsonoo can easily compose `decoder` or `encoder` functions to handle complex JSON values. The `Jsonoo.t` type is represented as a JS value and uses the Js_of_ocaml library to convert between OCaml values and JavaScript JSON values.

For example, to try to decode a list of integers, returning a default value otherwise:

```ocaml
let decode json =
  let open Jsonoo.Decode in
  try_default [] (list int) json
```

Since jsonoo provides [`t_of_js` and `t_to_js` functions](https://mnxn.github.io/jsonoo/jsonoo/Jsonoo/index.html#compatibility-with-gen_js_api), it is also possible to use the JSON type with gen_js_api bindings:

```ocaml
val send_json : t -> Jsonoo.t -> unit [@@js.call]
```

jsonoo seems to work well for its purpose of a Js_of_ocaml JSON library, but [@rgrinberg](https://github.com/rgrinberg/) [brought up the point](https://discuss.ocaml.org/t/ann-jsonoo-0-1-0/6480/4) that this fragments the JSON libraries based on their underlying JSON implementation. For that reason, it may be worthwhile to look into [json-data-encoding](https://gitlab.com/nomadic-labs/json-data-encoding), an alternative that allows using the same API across different JSON representations.



## Promises

As an extension that primarily operates on the user-interface level, asynchronous operations through the [JS Promise API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) are very important for a smooth user experience. Creating bindings to the promise functions seems straightforward at first, but you will eventually find that JS promises have a soundness problem.

For example, with a direct binding to the `resolve` function, one would expect that for every value passed to the function it would return that value wrapped in a promise.

```ocaml
val resolve : 'a -> 'a promise
```

```ocaml
let x : int promise = resolve 1
let y : int promise promise = resolve (resolve 2) (* flattened! *)
```

Everything seems fine from the OCaml side, but it turns out that JavaScript automatically flattens nested promises by following the `then` function of any value that is passed to it. Even though `y` appears to have the `int promise promise` type, the JS representation will be flattened to `int promise`. This is obviously a bad sign because the type system is misrepresenting the data, which will surely result in nasty runtime errors.

So how do we prevent the promise functions from following a value's `then` functions? The solution is simple: ensure that the JS functions never receive values that have a `then` function in the first place.

Using a technique I first saw in [@aantron](https://github.com/aantron)'s [promise library](https://github.com/aantron/promise) for BuckleScript, it is possible to check for values that have a `then` function and wrap them in an object to prevent the JS functions from calling it:

```javascript
function IndirectPromise(promise) {
	this.underlying = promise;
}

function wrap(value) {
	if (
		value !== undefined &&
		value !== null &&
		typeof value.then === "function"
	) {
		return new IndirectPromise(value);
	} else {
		return value;
	}
}

function unwrap(value) {
	if (value instanceof IndirectPromise) {
		return value.underlying;
	} else {
		return value;
	}
}
```

Calling `wrap` on every value that is passed to `Promise.resolve` and calling `unwrap` on every resolved (completed) value will make the behavior consistent. Doing the wrapping and unwrapping for each unsound promise function binding will result in an API that is suitable for type-safe usage.

The final product of these bindings is the [promise_jsoo](https://github.com/mnxn/promise_jsoo) ([opam](https://opam.ocaml.org/packages/promise_jsoo/)) library for Js_of_ocaml. It includes bindings for the majority of the JS API, as well as supplemental functions that make it easier to interoperate with OCaml. promise_jsoo provides the necessary functions to use it with gen_js_api. The documentation for promise_jsoo is available [here](https://mnxn.github.io/promise_jsoo/promise_jsoo/Promise/index.html).

As an added bonus, running an OCaml version of at least 4.08 (which wasn't possible with BuckleScript) allows using [binding operators](https://ocaml.org/releases/4.11/htmlman/bindingops.html):

```ocaml
val let* : 'a promise -> ('a -> 'b promise) -> 'b promise
```

```ocaml
let async_function () : int promise =
  let* first_num = get_num () in
  let* second_num = get_num () in
  async_calculation first_num second_num
```

This syntax is reminiscent of `await` syntax in other languages and it is a good alternative to the previous style of monadic operators:

```ocaml
let async_function () : int promise =
  get_num () >>= fun first_num ->
  get_num () >>= fun second_num ->
  async_calculation first_num second_num
```

promise_jsoo has a comprehensive test suite that should give weight to its claims of type safety. It was difficult to find a testing library that would work for an asynchronous library in Js_of_ocaml, but I eventually found [webtest](https://github.com/johnelse/ocaml-webtest) by [@johnelse](https://github.com/johnelse).

In the future, I'd like to investigate giving types to promise rejections and providing a simple way to convert to [Async](https://github.com/janestreet/async) or [Lwt](https://github.com/ocsigen/lwt) types.



## Sys.unix

Apparently, the value of `Sys.unix` in Js_of_ocaml is always true. The system seems to be hardcoded in Js_of_ocaml's [JS runtime](https://github.com/ocsigen/js_of_ocaml/blob/master/runtime/sys.js), which caused problems for path handling with the `Filename` OCaml module on a certain operating system (sorry Windows users!).

I assume the reason for the hardcoded system is because of a lack of a good way to get the operating system across different runtimes (browser, node.js). The browser has user agents and node.js has `process.platform`, but not vice versa.

As a workaround, the VSCode extension just uses bindings to the node [`path` module](https://nodejs.org/api/path.html) for proper cross-platform path handling since the extension already depends on node.js.



## Closing

Overall, I am very happy with the transition to Js_of_ocaml. The ability to use the same build system and packages for native and JS projects leads to a smooth and enjoyable development experience. I am still learning the quirks of the JS target, but for the most part, Js_of_ocaml *just works*.

The VSCode OCaml Platform is an actively developed project with numerous contributors, so please feel free to [submit an issue](https://github.com/ocamllabs/vscode-ocaml-platform/issues) or [contribute a pull-request](https://github.com/ocamllabs/vscode-ocaml-platform/pulls).

If you have any questions or comments about this post, I can answer them on the [OCaml forum topic](https://discuss.ocaml.org/t/js-of-ocaml-in-the-vscode-ocaml-platform/6635).
