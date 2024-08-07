# JavaScript Tooling

[`jco`](https://github.com/bytecodealliance/jco) is a fully native JS tool for working with the
emerging WebAssembly Components specification in JavaScript.

[Typescript][ts] can *also* be used, given that it is transpiled to JS first by relevant tooling (`tsc`).
`jco` includes a `jco types` subcommand for generating typings that can be used with a Typescript codebase.

[ts]: https://typescriptlang.org

## Installing `jco`

Installing [`jco`][jco] (and its required peer dependency [`componentize-js`][componentize-js]) can be done via NodeJS project tooling:

```console
npm install -g @bytecodealliance/componentize-js @bytecodealliance/jco
```

> [!NOTE]
> `jco` and `componentize-js` can be installed in a project-local manner with `npm install -D`

[ComponentizeJS][componentize-js] provides tooling used by `jco` to transpile JS to Wasm, so installing both packages is required.

[jco]: https://github.com/bytecodealliance/jco
[componentize-js]: https://github.com/bytecodealliance/ComponentizeJS

## Building a Component with `jco`

The first step in building a WebAssembly component is creating or downloading the interface that defines what your component can do. This usually means creating or downloading the [WebAssembly Interface Types ("WIT")][wit] world you would like to "target" with your component.

The [`adder` tutorial example](https://github.com/bytecodealliance/component-docs/tree/main/component-model/examples/tutorial/adder) will use the [`component` world](https://github.com/bytecodealliance/component-docs/tree/main/component-model/examples/tutorial/wit/adder/world.wit) which exports an `add` interface:

```wit
package docs:adder@0.1.0;

interface add {
    add: func(a: u32, b: u32) -> u32;
}

world adder {
    export add;
}
```

> [!NOTE]
> `export`ing the `add` interface meants that runners of the WebAssembly binary will be able to *call* that function.
>
> To learn more about the WIT syntax, check out the full [WIT specification][wit]

Along with this WIT interface, we can write a JavaScript module that implements the exported `add` function in the `adder` world:

```js
export const add = {
  add(x, y) {
    return x + y;
  }
};
```

> [!NOTE]
> In the code above, the `add` interface is exported, and the `add` function *inside* that interface is implemented.

With the WIT and Javascript in place, we can use [`jco`][jco] to create a WebAssembly component from the JS module, using `jco componentize`.

Our component is *so simple* (reminiscent of [Core WebAssembly][wasm-core], which deals primarily in numeric values) that we're actually *not using* any of the [WebAssembly System Interface][wasi] -- this means that we can `--disable` it when we invoke `jco componentize`

```console
jco componentize \
    --wit path/to/adder/world.wit \
    --world-name adder \
    --out add.wasm \
    --disable all \
    path/to/add.js
```

You should see output like the following:

```
OK Successfully written add.wasm.
```

> [!NOTE]
> As the `add` example is a regular [NodeJS][nodejs] project, you can run `npm install && npm run build`
> without having `jco` and `componentize-js` installed globally.

To run the component, we can use the [`example-host` project][example-host] (also used  in the [Rust `add` host](./rust.md#creating-a-command-component-with-cargo-component) instructions):

```console
cd component-model/examples/example-host
cargo run --release -- 1 2 ../path/to/add.wasm
1 + 2 = 3
```

> [!WARNING]
> You can run the above commands with `npm start`.
>
> Note that the code above depends on the [`example-host` Rust project][example-host], so the Rust toolchain is required for running that example, in particular [`cargo`][cargo]

While the output isn't exciting, the code contained in `example-host` does a lot to make it happen:

- Loads the WebAssembly binary at the provided path (in the command above, `../path/to/add.wasm`)
- Calls the `export`ed `add` function with arguments
- Prints the result

The important Rust code looks like this:

```rust
let component = Component::from_file(&engine, path).context("Component file not found")?;

let (instance, _) = Example::instantiate_async(&mut store, &component, &linker)
    .await
    .context("Failed to instantiate the example world")?;

instance
    .call_add(&mut store, x, y)
    .await
    .context("Failed to call add function")
```

A quick reminder on the power and new capabilities afforded by WebAssembly -- we've written, loaded, instantiated and executed Javascript from Rust with a strict interface, without the need for FFI, subprocesses or a network call.

[example-host]: https://github.com/bytecodealliance/component-docs/tree/main/component-model/examples/example-host
[wit]: https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md
[nodejs]: https://nodejs.org/en
[cargo]: https://doc.rust-lang.org/cargo
[wasi]: https://wasi.dev/
[wasm-core]: https://webassembly.github.io/spec/core/

## Running a Component from JavaScript Applications (including the Browser)

JavaScript runtimes available in browsers cannot yet execute WebAssembly components, a WebAssembly component
built in any language (Javascript or otherwise) must be transpiled into a JavaScript wrapper and
a [WebAssembly core module][wasm-core-module] which *can* be run by in-browser Wasm runtimes.

The Javascript toolchain also allows us to run WebAssembly binaries built in Javascript (or other languages) from Javascript-native projects.

Given an existing WebAssembly component (e.g. `add.wasm`), we can "transpile" the component into runnable Javscript by using `jco tranpsile`:

```console
npx jco transpile add.wasm -o out-dir
```

You would see output similar to the following:

```
  Transpiled JS Component Files:

 - out-dir/add.core.wasm                   10.1 MiB
 - out-dir/add.d.ts                         0.1 KiB
 - out-dir/add.js                          1.57 KiB
 - out-dir/interfaces/docs-adder-add.d.ts  0.08 KiB
```

> [!NOTE]
> To follow along, see the [`jco` example `add` component](https://github.com/bytecodealliance/jco/tree/main/examples/components/add).
>
> With the project pulled locally, you also run `npm run transpile` which outputs to `dist/transpiled`

Thanks to `jco` transpilation, you can import the resulting `out-dir/add.js` file and run it from any JavaScript application
using a runtime that supports the [core WebAssembly specification][core-wasm] as implemented for Javascript.

To use this component from [NodeJS][nodejs], you can write code like the following:

```mjs
import { add } from "./out-dir/add.mjs";

console.log("1 + 2 = " + add(1, 2));
```

Note that just like any other NdoeJS project, the [The `jco` project's `add` example component](https://github.com/bytecodealliance/jco/tree/main/examples/components/add)
makes use of build automation in `package.json`.

Now that we have NodeJS compatible Javascript that uses the transpiled component, you can execute it directly with `node`, and you should see the following output:

```
1 + 2 = 3
```

This is directly comparable to the Rust host code mentioned in the previous section. The Javascript code *also* runs the WebAssembly binary (which could have been written in any language) from the host language
(Javascript) natively.

> [!NOTE]
> Given WebAssembly's nature as a cross-language binary format, you can run
> a WebAssembly core module (or component) produced by Javascript from *any*
> language that supports using a WebAssembly runtime (ex. `wasmtime`)

[wasm-core-module]: https://webassembly.github.io/spec/core/binary/modules.html
[core-wasm]: https://webassembly.github.io/spec/core/

## Building Reactor components with `jco`

Reactor components are WebAssembly components that are long running and meant to be called repeatedly over time. They're analogous to libraries of functionality rather than an executable (a "command" component).

Reactor components (and components in general) expose their interfaces via [WebAssembly Interface Types][docs-wit], hand-in-hand with the [Component Model][docs-component-model] which enables components to use higher level types interchangably.

[docs-wit]: ../design/wit.md
[docs-component-model]: ../design/why-component-model.md

### Exporting WIT interfaces with `jco`

Packaging reusable functionality into WebAssembly components isn't useful if we have no way to *expose* that functionality. This section offers a slightly deeper dive into the usage of WIT in WebAssembly components that can use the Component Model.

As in the previous example, `export`ing WIT interfaces for other components (or a WebAssembly host) to use is fundamental to developing WebAssembly programs.

Let's examine a [`jco` example project called `string-reverse`][jco-examples-string-reverse] that exposes functionality for reversing a string.

To build a project like `string-reverse` from the ground up, first we'd start with a WIT like the following:

```wit
package example:string-reverse@0.1.0

@since(version = 0.1.0)
interface reverse {
    reverse-string: func(s: string) -> string;
}

world string-reverse {
    export reverse;
}
```

As a slightly deeper crash course on [WIT][wit], here's what the above code describes:

- We've defined a namespace called `examples`
- We've defined a package called `rev` inside the `examples` namespace
- This WIT file corresponds to version `0.1.0` of `example:rev` package
- We've defined an interface called `reverse` which contains *one* function called `reverse-string`
- We specify that the `reverse` interface has existed *since* the `0.1.0` version
- The `reverse-string` function (AKA. `example:reverse-string/reverse.reverse-string`) takes a string and returns a string
- We've defined a `world` called `component` which exports the functionality provided by the `reverse` interface

> [!WARNING]
> How do we *know* that `reverse` actually reverses a string?
>
> Unfortunately, that problem is not really solvable at this level -- this is between you and the writer of the component that implements the WIT interface.
>
> Of course, with WebAssembly, you *can* enforce static checks if you're so inclined, *before* you run any given binary.

OK now let's see what the JS code looks like to *implement* the `component` world:

```js
/**
 * This module is the JS implementation of the `string-reverse` WIT world
 */

/**
 * This Javascript will be interpreted by `jco` and turned into a
 * WebAssembly binary with a single export (this `reverse` function).
 */
function reverseString(s) {
  return s.reverse();
}

/**
 * The Javascript export below represents the export of the `reverse` interface,
 * which which contains `reverse-string` as it's primary exported function.
 */
export const reverse = {
    reverseString,
};
```

> [!NOTE]
> To view the full code listing along with instructions, see the [`examples/tutorials/jco/string-reverse` folder][jco-examples-string-reverse]

To use `jco` to compile this component, you can run the following from the `string-reverse` folder:

```console
npx jco componentize \
    string-reverse.mjs \
    --wit wit/component.wit \
    --world-name component \
    --out string-reverse.wasm \
    --disable all
```

> [!NOTE]
> Like the previous example, we're not using any of the advanced [WebAssembly System Interface][wasi] features, so we `--disable` all of them
>
> Rather than typing out the `jco componentize` command manually, you can also run
> the build command with [`npm run build` from the `string-reverse` folder](https://github.com/bytecodealliance/jco/blob/main/examples/components/string-reverse/package.json#L6).

You should see output like the following:

```
OK Successfully written string-reverse.wasm.
```

Now that we have a WebAssembly binary, we can *also* use `jco` to run it in a native Javascript context by *transpiling* the WebAsssembly binary (which could have come from anywhere!) to a Javascript module.

```console
npx jco transpile string-reverse.wasm -o dist/transpiled
```

You should see the following output:

```
  Transpiled JS Component Files:

 - dist/transpiled/interfaces/example-string-reverse-reverse.d.ts   0.1 KiB
 - dist/transpiled/string-reverse.core.wasm                        10.1 MiB
 - dist/transpiled/string-reverse.d.ts                             0.15 KiB
 - dist/transpiled/string-reverse.js                               2.55 KiB
```

> [!TIP]
> Yes, transpilation *does* produce [Typescript declaration file][ts-decl-file], so you can also use a Typescript-focused workflows.
>
> You can also run this `jco` example project step with `npm run transpile`, and the transpilation will happen, along with the rename.

Now that we have a transpiled module, we can run it from any Javascript context that supports core WebAssembly (whether NodeJS or the browser).

For NodeJS, we can use code like the following:

```mjs
// If this import listed below is missing, please run `npm run transpile`
import { reverse } from "./dist/transpiled/string-reverse.mjs";

const reversed = reverse.reverseString("!dlroW olleH");

console.log(`reverseString('!dlroW olleH') = ${reversed}`);
```

> [!NOTE]
> In the `jco` example project, you can run `npm run transpiled-js` to build the existing code.

Assuming you have the `dist/transpiled` folder populated (by running `jco transpile` in the previous step), you should see output like the following:

```
reverseString('!dlrow olleh') = hello world!
```

While it's somewhat redundant in this context, what we've done from NodeJS demonstrates the usefulness of WebAssembly and the `jco` toolchain. With the help of `jco`, we have:

- Compiled Javascript to a WebAssembly module (`jco compile`), adhering to an interface defined via WIT
- Converted the compiled WebAssembly module (which could be from *any* language) to a module that can be used from any compliant JS runtime (`jco transpile`)
- Run the transpiled WebAssembly component from a Javascript native runtime (NodeJS)

[repo]: https://github.com/bytecodealliance/component-docs
[jco-examples-string-reverse]: https://github.com/bytecodealliance/jco/tree/main/examples/components/string-reverse
[ts-decl-file]: https://www.typescriptlang.org/docs/handbook/declaration-files/deep-dive.html#declaration-file-theory-a-deep-dive

### Advanced: Importing and reusing WIT interfaces via composition

Just as `export`ing functionality is core to building useful WebAssembly components, and similarly `import`ing and reusing functionality is key to using the strengths of WebAssembly.

Restated, **WIT and the Component Model enable WebAssembly to *compose***. This means we can build on top of functionality that already exists and `export` *new* functionality that depends on existing functionality.

Let's say in addition to the reversing the string (in the previous example) we want to build shared functionality that *also* upper cases the text it receives.

We can reuse the reversing functionality *and* export a new interface which enables us to reverse and upper-case.

Here's the WIT to make that happen:

```wit
package example:string-reverse-upper@0.1.0;

@since(version = 0.1.0)
interface reversed-upper {
    reverse-and-uppercase: func(s: string) -> string;
}

world revup {
    //
    // NOTE, the import below translates to:
    // <namespace>:<package>/<interface>@<package version>
    //
    import example:string-reverse/reverse@0.1.0;

    export reversed-upper;
}
```

This time, the `world` named `revup` that we are building *relies* on the interface `reverse` in the package `string-reverse` from the namespace `example`.

We can make use of *any* WebAssembly component that matches that interface, as long as we *compose* their functionality with the component that implements the `revup` world.

The `revup` world `import`s (and makes use) of `reverse` in order to `export` (provide) the `reversed-upper` interface, which contains the `reverse-and-uppercase` function (in JS, `reverseAndUppercase`).

> [!NOTE]
> Functionality is imported from the `interface`, *not* the `world`. `world`s can be included/used, but the syntax is slightly different for that.

The Javascript to make this work ([`string-reverse-upper.mjs` in the repo](https://github.com/bytecodealliance/jco/blob/main/examples/components/string-reverse-upper/string-reverse-upper.mjs)) looks like this:

```js
/**
 * This module is the JS implementation of the `revup` WIT world
 */

/**
 * The import here is *virtual*. It refers to the `import`ed `reverse` interface in component.wit.
 *
 * These types *do not resolve* when the first `string-reverse-upper` component is built,
 * but the types are relevant for the resulting *composed* component.
 */
import { reverseString } from 'example:string-reverse/reverse@0.1.0';

/**
 * The Javascript export below represents the export of the `reversed-upper` interface,
 * which which contains `revup` as it's primary exported function.
 */
export const reversedUpper = {
  /**
   * Represents the implementation of the `reverse-and-uppercase` function in the `reversed-upper` interface
   *
   * This function makes use of `reverse-string` which is *imported* from another WebAssembly binary.
   */
  reverseAndUppercase() {
    return reverseString(s).toLocaleUpperCase();
  },
};
```

We can build the component with `jco componentize`:

```console
npx jco componentize \
    string-reverse-upper.mjs \
    --wit wit/ \
    --world-name revup \
    --out string-reverse-upper.incomplete.wasm \
    --disable all
```

While we've successfully built a WebAssembly component, unlike the other examples, ours is *not yet complete*.

We can see that if we print the WIT of the generated component by running `jco wit`:

```console
npx jco wit string-reverse-upper.incomplete.wasm
```

You should see output like the following:

```
package root:component;

world root {
  import example:string-reverse/reverse@0.1.0;

  export example:string-reverse-upper/reversed-upper@0.1.0;
}
```

This tells us that the component still has *unfulfilled `import`s* -- we *use* the `reverseString` function that's in `reverse` as if it exists, but it's not yet a real part of the WebAssembly component (hence we've named it `.incomplete.wasm`.

To compose the two components (`string-reverse-upper/string-reverse-upper.incomplete.wasm` and `string-reverse/string-reverse.wasm` we built earlier), we'll need the [WebAssembly Composition tool (`wac`)][wac]. We can use `wac plug`:

```console
wac plug \
    -o string-reverse-upper.wasm \
    --plug ../string-reverse/string-reverse.wasm \
    string-reverse-upper.incomplete.wasm
```

> [!NOTE]
> You can also run this step with `npm run compose`.

A new component `string-reverse-upper.wasm` should now be present, which is a "complete" component -- we can check the output of `jco wit` to ensure that all the imports are satisfied:

```wit
package root:component;

world root {
  export example:string-reverse-upper/reversed-upper@0.1.0;
}
```

It's as-if we never imported any functionality at all -- the functionality present in `string-reverse.wasm` has been *merged into* `string-reverse-upper.wasm`, and it now simply `export`s the advanced functionality.

We can run this completed component with in any WebAssembly-capable native Javascript environment by using a the transpiled result:

```console
npx jco transpile string-reverse-upper.wasm -o dist/transpiled
```

> [!NOTE]
> In the example project, you can run `npm run transpile` instead, which will also change the extension on `dist/transpiled/string-reverse-upper.js` to `.mjs`

You should see output like the following:

```
  Transpiled JS Component Files:

 - dist/transpiled/interfaces/example-string-reverse-upper-reversed-upper.d.ts  0.12 KiB
 - dist/transpiled/string-reverse-upper.core.wasm                               10.1 MiB
 - dist/transpiled/string-reverse-upper.core2.wasm                              10.1 MiB
 - dist/transpiled/string-reverse-upper.d.ts                                    0.19 KiB
 - dist/transpiled/string-reverse-upper.js                                      6.13 KiB
```

> [!TIP]
> Notice that there are *two* core WebAssembly files? That's because two core WebAssembly modules were involved
> in creating the ultimate functionality we needed.

To run the transpiled component, we can write code like the following:

```mjs
/**
 * If this import listed below is missing, please run
 *
 * ```
 * npm run build && npm run compose && npm run transpile`
 * ```
 */
import { reversedUpper } from "./dist/transpiled/string-reverse-upper.mjs";

const result = reversedUpper.reverseAndUppercase("!dlroW olleH");

console.log(`reverseAndUppercase('!dlroW olleH') = ${result}`);
```

> [!NOTE]
> In the [`jco` example project](https://github.com/bytecodealliance/jco/tree/main/examples/components/string-reverse-upper), you can run `npm run transpiled-js`

You should see output like the following:

```
reverseAndUppercase('!dlroW olleH') = HELLO WORLD!
```

[wac]: https://github.com/bytecodealliance/wac
