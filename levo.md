---
title: "Web: The Good Parts"
author:
  - "[Dimchikkk](https://github.com/Dimchikkk)"
  - "[tigregalis](https://github.com/tigregalis)"
---

## In the beginning

If you could peer through a portal into a parallel universe, you might see an alternate timeline where the Web evolved in a very different way. No HTML, no CSS, and no JavaScript: a Web built from the ground up for **applications**, not _documents_.

We've been there before, sort of. Flash, Java Applets, Silverlight. But those technologies had some very serious shortcomings. (Let's not get into that.)

All over the internet, you hear a common refrain from developers in opposite camps; those forced to develop for the Web, and those who refuse to: "Why is web development so damn complicated?" Well, what's the alternative? I think that endless argument is perfectly illustrated by this interaction.

[![The web is too complicated and the answer is native applications?](./tweet.PNG)](https://twitter.com/tsoding/status/1731311044637970446)

Are native apps really the way to go? What about shareability, universality, cross-platform compatibility, ease of access and ease of distribution? The hyperlink was the killer feature of the early Web, for a very good reason.

The fact is, the Web won. And let's be real, there is a subset of the Web platform that _is_ actually good.

## Born of the Web, will be its killer

As the modern Web has evolved away from simple websites ("documents") towards rich, interactive and highly-connected experiences ("applications"), the technology has evolved to meet those new demands.

Enter WebAssembly: a way to run secure sandboxed code at near-native speeds in the browser and **beyond**. But this is not an introduction to WASM, you can read up on that in your own time. You're here with us to explore the alternate universe of a Web built for applications.

Imagine running native-like interactive graphical applications that are sandboxed, with security through capabilities.

Allow us to introduce you to [levo](https://github.com/velostudio/levo), which is not a _browser_, but a **portal**.

_Full disclaimer: This was built over the holiday break, and in its current state it's more of a proof of concept than a fully-fledged platform that will ultimately dethrone the browser as the undisputed ruler of the Web._

## The elevator pitch

Portals will run your guest applications, written in any language, to allow a truly native desktop app experience that is shareable via URL.

## Show, don't tell

<video alt="A guest application being hosted by the portal" src="https://velostudio.github.io/blog/demo1.mp4" controls="controls"></video>

The video above (press Play!) shows a "guest" application, being "hosted" by the portal.

## Okay, do tell

The portal currently expects brotli-encoded `wasm32-wasi` binaries that meets the relevant `spec/host.wit` contract and is served over the network on a URL.

The WIT (WebAssembly Interface Types) format is an interface description language. We use it to define the contract between guest (client app) and host (portal): the guest `import`s functionality from the host, and `export`s a set of functions that the host can call:

```wit
// spec/host.wit

package levo:portal;

interface my-imports {

  variant mouse-button {
    // The left mouse button.
    left,
    // The right mouse button.
    right,
    // The middle mouse button.
    middle,
    // Another mouse button with the associated number.
    other(u16),
  }

  // ... other types

  record position {
    x: float32,
    y: float32,
  }

  record size {
    width: float32,
    height: float32,
  }

  label: func(text: string, x: float32, y: float32, size: float32, color: string);
  link: func(url: string, text: string, x: float32, y: float32, size: float32);
  delta-seconds: func() -> float32;
  key-just-pressed: func(key: key-code) -> bool;
  key-pressed: func(key: key-code) -> bool;
  key-just-released: func(key: key-code) -> bool;
  mouse-button-just-pressed: func(btn: mouse-button) -> bool;
  mouse-button-just-released: func(btn: mouse-button) -> bool;
  mouse-button-pressed: func(btn: mouse-button) -> bool;
  cursor-position: func() -> option<position>;
  canvas-size: func() -> size;
  // ... other functions
}

world my-world {
  import my-imports;

  export update: func();

  export setup: func();
}
```

You can use [`wit-bindgen-cli`](https://github.com/bytecodealliance/wit-bindgen/tree/main#cli-installation) to generate bindings for `spec/host.wit` for your chosen language, or at least one that compiles to `wasm32-wasi` and [is supported by `wit-bindgen`](https://github.com/bytecodealliance/wit-bindgen/tree/main#supported-guest-languages) (sorry Haskellers).

```bash
wit-bindgen tiny-go ../../spec --out-dir=my-world
```

Or, if you write your client app in [a better language](https://www.rust-lang.org/), you can use the `wit-bindgen` crate to generate bindings with the `wit_bindgen::generate!()` macro.

From your client app, you export a `setup()` function and an `update()` function, as defined in the exports of `spec/host.wit`. The example below is using Rust.

```rust
// src/lib.rs
use levo::portal::my_imports::*;

impl Guest for MyWorld {
    fn setup() {
        // canvas_size() is exposed by the host
        let size = canvas_size();
        let width = size.width;
        let height = size.height;
        let message = format!("Hello from Rust! ({width}x{height})");
        // print() is exposed by the host
        print(message);
        // thanks to WASI, and wasmtime providing an implementation of it,
        // guests can use parts of their standard library
        println!("Thank you WASI!");
    }
    fn update() {}
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]
```

We can write the same app in Go:

```go
// src/my-component.go
// bindings are in: my-world/
func (e HostImpl) Setup() {
        // worldLevoPortalMyImportsCanvasSize() is exposed by the host
        var width float32 = world.LevoPortalMyImportsCanvasSize().Width
        var height float32 = world.LevoPortalMyImportsCanvasSize().Height
        message := fmt.Sprintf("Hello from Go! (%dx%d)", width, height)
        // world.LevoPortalMyImportsPrint() is exposed by the host
        world.LevoPortalMyImportsPrint(message)
        // thanks to WASI, and wasmtime providing an implementation of it,
        // guests can use parts of their standard library
        fmt.Printf("Thank you WASI!");
}

func (e HostImpl) Update() {}
```

Compile your app as a C-compatible dynamic library to the target `wasm32-wasi` platform using your chosen language's build tools.

```bash
# Go
tinygo build -target=wasi -o main.wasm src/my-component.go

# Rust
cargo build --target wasm32-wasi --release
```

Once you've compiled your app, you can use `wasm-tools` to adapt your wasm binary to the WebAssembly Component Model.

```bash
wasm-tools component new ../../target/wasm32-wasi/release/rust_client_app.wasm \
  -o my-component.wasm --adapt ../wasi_snapshot_preview1.reactor.wasm
```

Compress that final artifact using `brotli` (which can be done using the included `brotli-encoder` tool).

```bash
cargo run --package brotli-encoder --release \
  -- my-component.wasm "../../levo-server/public/rust.wasm"
```

Finally, serve the compressed artifact (which can be done using the included `levo-server` static file server).

```bash
cd levo-server
SERVER_CONFIG_FILE=./config.toml cargo r --release
# serves files in the levo-server/public directory
```

Run the portal

```bash
cargo run --release --package portal
```

and navigate to the location of the client app wasm file using the portal's address bar `http://localhost:8080/rust.wasm`.

## On the shoulders of giants

levo is currently built on top of:

- WIT and the WebAssembly Component Model: webassembly interface types, using `*.wit` files to define the contracts between the host portal and guest client applications
- `wasm-tools` and `wit-bindgen`: tooling that means we can make use of the above
- `WASI`: the ability to interact with the system securely, which means client apps can make use of their existing standard libraries
- [wasmtime](https://wasmtime.dev/): an embeddable WASM runtime with support for WASI and the component model
- `bevy`: a delightful game engine that builds on top of the incredible Rust ecosystem to handle everything from input to windowing to audio to GPU to rendering and much more

## The view from 30,000 feet

If you look at the [levo](https://github.com/velostudio/levo) repo, you'll find a few directories:

```
levo/               - you are here

    spec/           - this contains the host.wit
                      interface, which is the
                      real heart of the project

    portal/         - this contains the portal
                      (host) implementation

    clients/        - this contains the same
                      guest application built in
                      3 languages: Rust, Go and C

    brotli-encoder/ - this is a simple command
                      line tool to compress
                      the wasm binary using
                      brotli encoding

    levo-server/    - this is a simple server that
                      serves the brotli-encoded
                      wasm client applications
                      from a directory on the
                      file system
```

The current structure is _aspirational_. We're using cutting-edge technologies like WASM, WIT, and WASI, and we've structured this based on the assumption of guest applications being served over a network. We're very much building on top of quicksand here, but we know these technologies are the future of the Web, however they ultimately compose.

## Diving deeper

The `portal` app is the meat of the project, and is the first prototype implementation of a "portal".

It's built with [Bevy](https://bevyengine.org/),
which will give us easy access to `winit`, `wgpu` and
a host of other amazing projects in the Rust ecosystem,
parts of which we plan to expose through capabilities.

In its current form, it has an address bar that accepts a [URL]{.underline}
to the client app WASM file.
After entering the [URL]{.underline} and pressing Enter, the portal connects to the server,
downloads the brotli-encoded wasm file, decodes it, and initializes the `wasmtime`
runtime like so:

```rust
// Set up Wasmtime components
let mut config = Config::new();
config.wasm_component_model(true).async_support(false);
let engine = Engine::new(&config)?;
let component = Component::new(&engine, decoded_input)?;

// Set up Wasmtime linker
let mut linker = Linker::new(&engine);
sync::add_to_linker(&mut linker)?;
let table = Table::new();
let wasi = WasiCtxBuilder::new().build();
// levo::portal::MyWorld is generated by the wasmtime::bindgen!() macro
MyWorld::add_to_linker(&mut linker, |state: &mut MyCtx| state)?;
// Set up Wasmtime store
let mut store = Store::new(
    &engine,
    MyCtx {
        table,
        wasi,
        channel: cc,
    },
);
let (bindings, _) = MyWorld::instantiate(&mut store, &component, &linker)?;
```

The current API and implementation for the portal is simple:

1. it runs the client-provided [setup]{.underline} function once.
2. it runs the client-provided [update]{.underline} function every frame.

During those functions,
the guest (client app) can use any portal functions declared in the `spec/host.wit` file
and the host (the portal) provides implementations for these interfaces.

For example,

1. the client calls [begin-path]{.underline}, so the host begins a vector path
2. later, the client calls [cubic-bezier-to]{.underline}, so the host adds a cubic bezier curve to the current path
3. later, the client calls [close-path]{.underline}, so the host closes the current path
4. later, the client calls [fill]{.underline}, so the host
   finalizes the current path and fills it.

After the guest update schedule,
the host will call the corresponding `bevy_prototype_lyon` functions
to construct the entities and ultimately draw to the screen.

At the next frame, we despawn previous entities and start again. It's an immediate mode style API.

## You shall not pass

`wasmtime`{.verbatim} supports the **component model** and **wasi**:

> WASI is designed with capability-based security principles, using the
> facilities provided by the Wasm component model. All access to
> external resources is provided by capabilities.

This allows us to run guest code without risking the security of
the host machine, while limiting access to some capabilities.
As a proof of concept for capabilities, we allow a guest app to read from a single directory specified via the `--allow-read` command-line flag.
Take a look at [rust-test-read-file](https://github.com/velostudio/levo/tree/main/clients/rust-test-read-file).

```
cargo run --release --package portal -- --allow-read "./public"
```

Given this file tree:

```
public/
  hello.txt
private/
  secret.txt
```

```rs
// from client app
let Ok(hello) = levo::portal::my_imports::read_file("hello.txt") else {
    // this error will not print
    print("Failed to read public/hello.txt");
    return;
};
// the contents of `public/hello.txt` will print
print(&String::from_utf8_lossy(&hello));
let Ok(secret) = levo::portal::my_imports::read_file("../private/secret.txt") else {
    // this error will print
    print("Failed to read private/secret.txt");
    return;
};
// the contents of `private/secret.txt` will not print
print(&String::from_utf8_lossy(&secret));
```

If `--allow-read` is omitted, neither of the files will be read.  
If `--allow-read="./public"`, the contents of `./public/hello.txt` is printed successfully.

## A dream of Spring

We want to be able to make and meet this promise: write a native app in your chosen language that is secure and massively distributable.

`spec/host.wit` is the real heart of the project, and we want to explore the design of the APIs exposed by the portal. There is a tension between providing a high level API (the start of which we have now) and a low level API for more granular and powerful control of the host machine. We want to explore both, however, for now the focus is on the high level API so we get those clear wins.

Our next steps:

- Flesh out the high level API, both in the spec and in the portal implementation - being built on Bevy and exposing an immediate mode style API it's already in a state where you could implement 2D HTML-canvas style games

  - We could look to other projects for inspiration, such as macroquad, raylib, and others

- As an option, we could repurpose the portal shell to run in non-network contexts,

  - or we could look to Tauri for inspiration for embedding these applications
  - or we could make use of Bevy's Asset system for a cleaner development experience

- Investigate and implement a low level API to provide access to, or mirror, `wgpu`, `winit`, and so on, both in the spec and in the portal implementation - this would enable `fn main()` style apps where the guest app owns the loop - Bevy provides access to these ecosystems

  - The eventual goal here would be to make a "portal" be a platform target for guest applications from multiple languages
  - Imagine, for instance, compiling a Bevy game to run directly on the portal without changing any user code

- Build out the "capabilities" architecture in the portal implementation; we will look to Deno and Tauri here for inspiration. For example, being able to provide restrict access to resources such as

  - file system (see deno / tauri)
  - stdin / stdout (see deno / tauri)
  - network (see deno / tauri)
  - system time
  - memory budget
  - "CPU" budget
  - GPU / hardware acceleration
  - cryptography / random number generation
  - window management
  - audio
  - the URL of the page (route)

## The end (the beginning)

Let's take a step back and peer through the looking glass into a possible future.

HTML, CSS and JS are a thing of the past. There is a new standard. You choose to use **Rust** ([lib.rs](https://github.com/velostudio/levo/blob/main/clients/rust-client-app/src/lib.rs)), **C**
([my-component.c](https://github.com/velostudio/levo/blob/main/clients/c-client-app/src/my-component.c)),
**C++**, **Java**, **Go** ([my-component.go](https://github.com/velostudio/levo/blob/main/clients/go-client-app/src/my-component.go)) or some emerging future language
to write native-like app experiences that can be quickly and easily shared, distributed and executed securely.

## The once and future king

Perhaps one day the browsers will implement portals, or something like it, natively.

Until then, we'll just have to build it ourselves.
