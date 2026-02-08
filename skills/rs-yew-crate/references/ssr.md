# Server-Side Rendering (SSR) and Hydration

## Table of Contents

- [Overview](#overview)
- [Basic SSR Setup](#basic-ssr-setup)
- [Component Lifecycle During SSR](#component-lifecycle-during-ssr)
- [Hydration](#hydration)
- [Suspense and Data Fetching](#suspense-and-data-fetching)
- [Single Thread Mode (WASI)](#single-thread-mode-wasi)
- [SSR with Routing](#ssr-with-routing)
- [Best Practices](#best-practices)

## Overview

Server-side rendering generates HTML on the server, solving:

1. **First paint latency** — users see content before WASM loads
2. **SEO** — search engines index pre-rendered content

Yew provides `ServerRenderer` for rendering and `Renderer::hydrate()` for client-side hydration.

> **Status**: SSR is experimental as of Yew 0.22.

## Basic SSR Setup

```rust
use yew::prelude::*;
use yew::ServerRenderer;

#[component]
fn App() -> Html {
    html! { <div>{"Hello, World!"}</div> }
}

#[tokio::main]
async fn main() {
    let renderer = ServerRenderer::<App>::new();
    let html_string = renderer.render().await;
    println!("{}", html_string);
    // Output: <div>Hello, World!</div>
}
```

For rendering with props:

```rust
let props = AppProps { /* ... */ };
let renderer = ServerRenderer::<App>::with_props(move || props);
let html = renderer.render().await;
```

### Streaming Rendering

```rust
let renderer = ServerRenderer::<App>::new();
let mut body = String::new();
// render_stream provides a Stream of String chunks
renderer.render_stream().for_each(|chunk| {
    body.push_str(&chunk);
    async {}
}).await;
```

## Component Lifecycle During SSR

- All hooks **except** `use_effect` and `use_effect_with` run normally
- Effects are **not executed** during SSR
- Web APIs (`web_sys`, `window()`, `document()`) are **not available** — will panic

```rust
// SAFE during SSR — runs in effect only
use_effect(|| {
    // web_sys calls here — only runs on client
    let window = web_sys::window().unwrap();
    || ()
});

// DANGEROUS during SSR — runs in render
fn view() -> Html {
    let window = web_sys::window().unwrap(); // PANICS on server!
    html! {}
}
```

### Struct Components During SSR

- Lifecycle methods invoked in different order than client
- Messages accepted until all children rendered and `destroy` called
- No messages should trigger Web API logic
- **Prefer function components for SSR**

## Hydration

Hydration connects client-side Yew to server-rendered HTML. `ServerRenderer` produces hydratable HTML with extra metadata.

### Client-Side Hydration

```rust
use yew::prelude::*;
use yew::Renderer;

#[component]
fn App() -> Html {
    html! { <div>{"Hello, World!"}</div> }
}

fn main() {
    let renderer = Renderer::<App>::new();
    renderer.hydrate(); // connects to existing server-rendered DOM
}
```

### Hydration Rules

1. **Virtual DOM must match SSR output exactly** — including component tree structure
2. **Use `PhantomComponent`** for components only present in one rendering mode
3. **HTML must be spec-compliant** — browsers auto-correct invalid HTML (e.g., `<table>` without `<tbody>`), causing hydration mismatches

### Lifecycle During Hydration

- Components schedule 2 consecutive renders after creation
- Effects called after the second render completes
- Render functions must be side-effect free
- DOM considered disconnected until `rendered()` is called (struct components)

## Suspense and Data Fetching

Suspense enables render-as-you-fetch pattern. During SSR, Yew waits for suspended components to resolve before serializing.

### Suspense-Based Data Fetching

```rust
use yew::prelude::*;
use yew::suspense::{Suspension, SuspensionResult};

#[hook]
fn use_data() -> SuspensionResult<Data> {
    match try_load_data() {
        Some(data) => Ok(data),
        None => {
            let (suspension, handle) = Suspension::new();
            // When data loads, call handle.resume()
            on_data_ready(move || { handle.resume(); });
            Err(suspension)
        }
    }
}

// Component using suspense hook
#[component]
fn Content() -> HtmlResult {
    let data = use_data()?; // ? converts Err(Suspension) to Html
    Ok(html! { <div>{ &data.value }</div> })
}

// Parent with fallback
#[component]
fn App() -> Html {
    let fallback = html! { <div>{"Loading..."}</div> };
    html! {
        <Suspense {fallback}>
            <Content />
        </Suspense>
    }
}
```

### Suspension Handle

`Suspension::new()` returns `(Suspension, SuspensionHandle)`:

- Call `handle.resume()` to re-render suspended components
- Dropping the handle also resumes (triggers re-render)
- **Must store handle** until data is ready — dropping early causes infinite re-render loops

### SSR Behavior with Suspense

- Server waits for all suspended components to resolve
- During hydration, elements within `<Suspense>` remain dehydrated until children are no longer suspended
- Enables data fetching without repeated render cycles

## Single Thread Mode (WASI)

For WASI or single-threaded environments:

```rust
use yew::prelude::*;
use yew::LocalServerRenderer;

#[component]
fn App() -> Html {
    html! { <h1>{"Yew WASI SSR"}</h1> }
}

pub async fn render() -> String {
    let renderer = LocalServerRenderer::<App>::new();
    let html = renderer.render().await;

    let mut body = String::new();
    body.push_str("<body><div id='app'>");
    body.push_str(&html);
    body.push_str("</div></body>");
    body
}
```

Build with `wasm32-wasip1` or `wasm32-wasip2` target.

For `wasm32-unknown-unknown` SSR (e.g., Cloudflare Workers), use `not_browser_env` feature flag.

## SSR with Routing

Use `yew-router` with SSR by providing route context:

```rust
// Server side
let url = "/some/path".to_string();
let renderer = ServerRenderer::<App>::with_props(move || {
    ServerAppProps { url }
});
```

See `examples/ssr_router` in the Yew repository for a complete example.

## Best Practices

1. **Isolate Web API calls** in `use_effect` / `use_effect_with`
2. **Use function components** — struct components have complex SSR lifecycle
3. **Ensure HTML spec compliance** — avoid structures browsers auto-correct
4. **Keep render functions pure** — no state mutation, no additional renders
5. **Use Suspense for data fetching** — avoids repeated render cycles
6. **Test hydration** — mismatches between SSR and client rendering cause subtle bugs
7. **Consider streaming** for large pages to improve Time to First Byte
