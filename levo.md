---
title: 'Web: The Good Parts'
author:
  - '[StaffEngineer](https://github.com/StaffEngineer)'
  - '[tigregalis](https://github.com/tigregalis)'
---

## In the beginning

If you could peer through a portal into a parallel universe, you might see an alternative timeline where the Web evolved in a very different way. No HTML, no CSS, and no JavaScript: a Web built from the ground up for **applications**, not _documents_.

We've been there before, sort of. Flash, Java Applets, Silverlight. But those technologies had some very serious shortcomings. (Let's not get into that.)

All over the internet, you hear a common refrain from developers in opposite camps; those forced to develop for the Web, and those who refuse to: "Why is web development so damn complicated?"" Well, what's the alternative? I think that endless argument is perfectly illustrated by this interaction.

[![The web is too complicated and the answer is native applications?](tweet.png)](https://twitter.com/tsoding/status/1731311044637970446)

Are native apps really the way to go? What about shareability, universality, cross-platform compatibility, ease of access and ease of distribution? The hyperlink was the killer feature of the early Web, for a very good reason.

The fact is, the Web won. And let's be real, there is a subset of the Web platform that _is_ actually good.

## Born of the Web, will be its killer

As the modern Web has evolved towards rich, interactive and highly-connected experiences ("applications") rather than simple websites ("documents"), the technology has evolved to meet those new demands.

Enter WebAssembly: a way to run secure sandboxed code at near-native speeds in the browser and **beyond**. But this is not an introduction to WASM, you can read up on that in your own time. You're here with us to explore the alternative universe of a Web built for applications.

Imagine running native-like interactive graphical applications that are sandboxed, with security through capabilities.

Allow us to introduce you to [levo](https://github.com/velostudio/levo), which is not a _browser_, but a **portal**.

_Full disclaimer: This was built over the holiday break, and in its current state it's more of a proof of concept than a fully-fledged platform that will ultimately dethrone the browser as the undisputed ruler of the Web._

## The elevator pitch

Portals will run your guest applications, written in any language, to allow a truly native desktop app experience that is shareable via URL.

## Show, don't tell

<video src="https://velostudio.github.io/blog/demo1.mp4" controls="controls"></video>

Press play to see the video. A "guest" application, being "hosted" by the portal.

## The view from above

If you look at the [levo](https://github.com/velostudio/levo) repo, you'll find a few directories:

```
levo/               - you are here
    spec/           - this contains the host.wit interface, which is the real heart of the project
    portal/         - this contains the portal (host) implementation
    clients/        - this contains the same guest application built in 3 languages: Rust, Go and C
    brotli-encoder  - this is a simple command line tool to brotli-encode the wasm
    levo-server/    - this is a simple server that serves the brotli-encoded wasm client applications from the file system
```

The current structure and all of the dependencies in use is _aspirational_. We're using cutting-edge technologies like WASM, WIT, and WASI, and we've structured this based on the assumption of guest applications being served over a modern and future network. We're very much building on top of quicksand here, but we know these technologies are the future of the Web, however they ultimately compose.

## A walkthrough (okay, do tell)

The portal currently expects brotli-encoded `wasm32-wasi` binaries that meets the relevant `spec/host.wit` contract and is served over URL.

You write your client apps in your chosen language that compiles to WASM (specifically, `wasm32-wasi`), and you export a `setup()` function and an `update()` function, as defined in `spec/host.wit`. From within your app, you can access the host functions declared:

```wit
package levo:portal;

interface my-imports {
  ...
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
}

world my-world {
  import my-imports;

  export update: func();

  export setup: func();
}
```

Once you've compiled your app to `wasm32-wasi`, you can use `wasm-tools` to adapt your wasm binary to the WebAssembly Component Model.

```bash
wasm-tools component new ../../target/wasm32-wasi/release/rust_client_app.wasm \
  -o my-component.wasm --adapt ../wasi_snapshot_preview1.reactor.wasm
```

You compress that artifact using `brotli` (which can be done using the included `brotli-encoder` tool).

```bash
cargo run --package brotli-encoder --release -- my-component.wasm "../../levo-server/public/rust.wasm"
```

Serve the compressed artifact via `WebTransport` (which can be done using the included `levo-server` server). The `WebTransport` protocol allows quick transfer of client app to the portal using UDP
packets.

The `portal` app is the meat of the project. It's written in `Bevy`, which
gives us easy access to `winit`, `wgpu` and a host of other amazing integrations with the Rust ecosystem. At least, that's the promise. In its current form, it just has one text input field that accepts a
[URL]{.underline} to the WASM resource.

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

Then, it runs the client-provided **setup** function once, and its **update** function every
frame. During those functions, the client can use any host functions declared in the
`spec/host.wit` file. The host app (the portal) provides implementations for these interfaces: (e.g. if client calls the [cubic-bezier-to]{.underline} function - the
host will call the corresponding `bevy_prototype_lyon` function to draw
a cubic bezier curve to the screen.

`wasmtime`{.verbatim} uses **component model** and **wasi**:

> WASI is designed with capability-based security principles, using the
> facilities provided by the Wasm component model. All access to
> external resources is provided by capabilities.

This allows you to run client code without risking the security of
the host machine. For example the host app can allow only specific amount of memory for the guest app. Let's try to load rust guest app giving maximum 1MB a linear memory to grow.
Since it needs more than 1MB on startup it simply won't be loaded by the host app, but 2MB is enough:

<video src="https://velostudio.github.io/blog/demo2.mp4" controls="controls"></video>

Press play to see the video. Portal controls max "guest" memory size via `max_memory_mb` URL query.

## Standing on the shoulders of giants

levo is currently built on top of:

- WIT and the WebAssembly Component Model: webassembly interface types, using \*.wit files to define the contracts between the host portal and guest applications
- wasm-tools and wit-bindgen: tooling for the above
- wasmtime: an embeddable WASM runtime
- bevy: a game engine that builds on top of the incredible Rust ecosystem to handle everything from input to windowing to audio to GPU to rendering and more

## A dream of spring

We want to be able to make and meet this promise: write a native app in your chosen language that is secure and massively distributable.

`spec/host.wit` is the heart of the project, and we want to explore the design of the portal. There is a tension between providing a high level API (the start of which we have now) and a low level API for more granular and powerful control of the host machine. We want to explore both, however, the high level API is easier to develop.

Our next steps:

- Flesh out the high level API, both in the spec and in the portal implementation - being built on Bevy and exposing an immediate mode style API it's already in a state where you could implement 2D HTML-canvas style games - we could look to other projects for inspiration, such as macroquad, raylib, and others

- Break the web-native assumptions of an app served over the Web - as option, we could link a specific binary directly at startup to open N windows of app - or we could look to Tauri for inspiration for embedding these applications - or we could make use of Bevy's Asset system for a cleaner development experience

- Investigate and implement a low level API to provide access to, or mirror, `wgpu`, `winit`, and so on, both in the spec and in the portal implementation - this would enable `fn main()` style apps where the guest app owns the loop - Bevy provides access to these ecosystems

  - The eventual goal here would be to make a "portal" be a platform target for guest applications from multiple languages
  - Imagine, for instance, compiling a Bevy game to run directly on the portal without changing much or any user code

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

You choose to use **Rust** ([lib.rs](https://github.com/velostudio/levo/blob/main/clients/rust-client-app/src/lib.rs)), **C**
([my-component.c](https://github.com/velostudio/levo/blob/main/clients/c-client-app/src/my-component.c)),
**C++**, **Java**, **Go** ([my-component.go](https://github.com/velostudio/levo/blob/main/clients/go-client-app/src/my-component.go)) or some unknown future language
to write native-like app experiences that can be quickly and easily shared, distributed via HTTP3 and executed securely.

## The once and future king

Perhaps one day the browsers will implement something like Portals natively. Until then, we'll just have to build it ourselves.
