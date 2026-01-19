# Writing Plugins

Pruner plugins are WASM components that implement the `pruner/plugin-api` WIT interface. You can find the source for the
WIT interface [here](../wit/world.wit).

The interface is straight-forward requiring you to implement only a `format` function which takes in source code as a
byte-array and should return the formatted result as a byte-array.

If you are building this in rust, pruner exposes a [pruner-plugin-api](../crates/plugin-api/) crate containing all the
generated interfaces which you can reference without needing to know anything about WASM or WIT.

## Rust Example

Here is a short reference on how to write a plugin for Pruner in Rust. For this we will reimplement the
[trim-newlines](https://github.com/pruner-formatter/plugin-trim-newlines) plugin which makes for a perfect, simple
example. Head over to that repository to see the end result of this document.

First we need to setup the project and for that we need to create a `Cargo.toml` which can be compiled as a WASM
component. Really this just means it needs to be defined as a `cdylib` crate.

```toml
# Cargo.toml
[package]
name = "trim-newlines"
version = "0.0.1"
edition = "2024"
resolver = "2"

[dependencies]
pruner-plugin-api = "1"

[lib]
crate-type = ["cdylib"]
```

Next we should create a `lib.rs` file which defines and exports a `Component` struct that implements the `PluginApi`
trait from `pruner-plugin-api`.

```rust
// lib.rs
use pruner_plugin_api::{FormatError, FormatOpts, PluginApi};

struct Component;

impl PluginApi for Component {
  fn format(source: Vec<u8>, _opts: FormatOpts) -> Result<Vec<u8>, FormatError> {
    Ok(source)
  }
}

pruner_plugin_api::bindings::export!(Component);
```

And that's it for a functioning plugin! You can now compile this directly with cargo, making sure to target
`wasm32-wasip2`:

```bash
cargo build --release --target wasm32-wasip2
```

Which will output a compiled plugin to `target/wasm32-wasip2/release/trim_newlines.wasm` that is ready to be loaded up
and called by Pruner. Lets add it into our Pruner config:

```toml
# pruner.toml
[plugins]
trim_newlines = "file:///path/to/target/wasm32-wasip2/release/trim_newlines.wasm"

[languages]
markdown = ["trim_newlines"]
```

Running pruner with `--log-level=debug` should show your plugin being loaded and instantiated:

```bash
cat README.md | pruner format --lang markdown --log-level debug
```

It's not doing very much right now, so let's add some actual behaviour to this plugin:

```rust
// lib.rs
use pruner_plugin_api::{FormatError, FormatOpts, PluginApi};

struct Component;

impl PluginApi for Component {
  fn format(source: Vec<u8>, _opts: FormatOpts) -> Result<Vec<u8>, FormatError> {
    let mut start = 0;
    let mut end = source.len();

    while start < end && (source[start] == b'\n' || source[start] == b'\r') {
      start += 1;
    }

    while end > start && (source[end - 1] == b'\n' || source[end - 1] == b'\r') {
      end -= 1;
    }

    let mut result = source[start..end].to_vec();
    result.push(b'\n');
    Ok(result)
  }
}

pruner_plugin_api::bindings::export!(Component);
```

Recompile with `cargo build --release --target wasm32-wasip2` and we are done. Newlines will now be trimmed from our
markdown files!

## I want to write plugins in $LANGUAGE!

Theoretically anything that can compile to WASM components should be usable to write plugins for Pruner. There is no
official documentation in Pruner for how to do this - but I would be happy to accept contributions to that effect.

You can find some great WASM components documentation for other languages
[here](https://component-model.bytecodealliance.org/language-support.html) which should be enough to get going.
