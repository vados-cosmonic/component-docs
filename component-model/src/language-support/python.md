# Python Tooling

## Building a Component with `componentize-py`

[`componentize-py`](https://github.com/bytecodealliance/componentize-py) is a tool that converts a Python
application to a WebAssembly component.

First, install [Python 3.10 or later](https://www.python.org/) and [pip](https://pypi.org/project/pip/) if you don't already have them. Then, install `componentize-py`:

```sh
pip install componentize-py
```

Next, create or download the WIT world you would like to target. For this example we will use an [`example`
world](https://github.com/bytecodealliance/component-docs/tree/main/component-model/examples/example-host/add.wit) with an `add` function:

```wit
package example:component;

world example {
    export add: func(x: s32, y: s32) -> s32;
}
```

If you want to generate bindings produced for the WIT world (for an IDE or typechecker), you can generate them using the `bindings` subcommand. Specify the path to the WIT interface with the world you are targeting:

```sh
$ componentize-py --wit-path /path/to/examples/example-host/add.wit --world example bindings .
```

> You do not need to generate the bindings in order to `componentize` in the next step. `componentize` will generate bindings on-the-fly and bundle them into the produced component.

You can see that bindings were created in an `example` package which contains an `Example` protocol with an `add` method that we can implement:

```sh
$ cat<<EOT >> app.py
import example

class Example(example.Example):
    def add(self, x: int, y: int) -> int:
        return x + y
EOT
```

We now can compile our application to a Wasm component using the `componentize` subcommand:

```sh
$ componentize-py --wit-path /path/to/examples/example-host/add.wit --world example componentize app -o add.wasm
Component built successfully
```

To test the component, run it using the [Rust `add` host](./rust.md#creating-a-command-component-with-cargo-component):

```sh
$ cd component-model/examples/add-host
$ cargo run --release -- 1 2 ../path/to/add.wasm
1 + 2 = 3
```

See [`componentize-py`'s examples](https://github.com/bytecodealliance/componentize-py/tree/main/examples) to try out build HTTP, CLI, and TCP components from Python applications.


### Building a Component that Exports an Interface

The [sample `add.wit` file](https://github.com/bytecodealliance/component-docs/tree/main/component-model/examples/example-host/add.wit) exports a function. However, you'll often prefer to export an interface, either to comply with an existing specification or to capture a set of functions and types that tend to go together. Let's expand our example world to export an interface rather than directly export the function.

```wit
// add-interface.wit
package example:component;

interface add {
    add: func(a: u32, b: u32) -> u32;
}

world example {
    export add;
}
```

If you peek at the bindings, you'll notice that we now implement a class for the `add` interface rather than for the `example` world. This is a consistent pattern. As you export more interfaces from your world, you implement more classes. Our add example gets the slight update of:

```py
# app.py
import example

class Add(example.Example):
    def add(self, a: int, b: int) -> int:
        return a + b
```

Once again, compile an application to a Wasm component using the `componentize` subcommand:

```sh
$ componentize-py --wit-path add-interface.wit --world example componentize app -o add.wasm
Component built successfully
```

## Running components from Python Applications

Wasm components can also be invoked from Python applications. This walks through using tooling
to call the [`app.wasm` component from the examples](../../examples/example-host/add.wasm).

> `wasmtime-py` is only able to run components built with `componentize-py` when the `--stub-wasi` option is used at build time. This is because `wasmtime-py` does not yet support [resources](../design/wit.md#resources), and `componentize-py` by default generates components which use resources from the `wasi:cli` world.  See [this example](https://github.com/bytecodealliance/componentize-py/tree/main/examples/sandbox) of using the `--stub-wasi` option to generate a `wasmtime-py`-compatible component.

First, install [Python 3.11 or later](https://www.python.org/) and [pip](https://pypi.org/project/pip/) if you don't already have them. Then, install [`wasmtime-py`](https://github.com/bytecodealliance/wasmtime-py):

```sh
$ pip install wasmtime
```

First, generate the bindings to be able to call the component from a Python host application.

```sh
# Get an add component that does not import the WASI CLI world
$ wget https://github.com/bytecodealliance/component-docs/raw/main/component-model/examples/example-host/add.wasm
$ python3 -m wasmtime.bindgen add.wasm --out-dir add
```

The generated package `add` has all of the requisite exports/imports for the
component and is annotated with types to assist with type-checking and
self-documentation as much as possible. Inside the package is a `Root` class
with an `add` function that calls the component's exported `add` function. We
can now write a Python program that calls `add`:

```py
from add import Root
from wasmtime import Store

def main():
    store = Store()
    component = Root(store)
    print("1 + 2 = ", component.add(store, 1, 2))

if __name__ == '__main__':
    main()
```

Run the Python host program:

```sh
$ python3 host.py
1 + 2 = 3
```
