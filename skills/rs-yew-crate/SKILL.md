---
name: rs-yew-crate
description: Expert guidance for building Rust + WebAssembly frontend web applications using the Yew framework (v0.22). Use when creating, modifying, debugging, or architecting Yew applications — including function components, hooks, props, routing, contexts, events, server-side rendering, agents, and Suspense. Covers project setup with Trunk, the html! macro, state management, data fetching, and integration with the broader Yew/WASM ecosystem (yew-router, gloo, wasm-bindgen, web-sys, stylist, yewdux).
---

# Yew Framework (v0.22)

Yew is a modern Rust framework for creating multi-threaded frontend web applications compiled to WebAssembly. It uses a virtual DOM, component-based architecture, and a JSX-like `html!` macro. MSRV: Rust 1.84.0.

## Project Setup

### Prerequisites

```bash
rustup target add wasm32-unknown-unknown
cargo install --locked trunk
```

### New Project (manual)

```bash
cargo new yew-app && cd yew-app
```

**Cargo.toml**:

```toml
[package]
name = "yew-app"
version = "0.1.0"
edition = "2021"

[dependencies]
yew = { version = "0.22", features = ["csr"] }
```

> Feature `csr` enables `Renderer` and client-side rendering code. Omit for library crates. Use `ssr` for server-side rendering. Use `serde` for serde integration on `AttrValue`.

**index.html** (project root):

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Yew App</title>
  </head>
  <body></body>
</html>
```

**src/main.rs**:

```rust
use yew::prelude::*;

#[component]
fn App() -> Html {
    let counter = use_state(|| 0);
    let onclick = {
        let counter = counter.clone();
        move |_| { counter.set(*counter + 1); }
    };

    html! {
        <div>
            <button {onclick}>{ "+1" }</button>
            <p>{ *counter }</p>
        </div>
    }
}

fn main() {
    yew::Renderer::<App>::new().render();
}
```

```bash
trunk serve --open
```

### Starter Template

```bash
cargo generate --git https://github.com/yewstack/yew-trunk-minimal-template
trunk serve
```

### Trunk Configuration (Trunk.toml)

```toml
[serve]
address = "127.0.0.1"
port = 8080
```

## Function Components

The recommended component model. Annotate with `#[component]`, name in PascalCase, return `Html`.

```rust
use yew::{component, html, Html};

#[component]
fn HelloWorld() -> Html {
    html! { "Hello world" }
}

// Usage:
html! { <HelloWorld /> }
```

### Properties (Props)

Derive `Properties + PartialEq`. Use `AttrValue` instead of `String` for attribute values (cheap to clone).

```rust
use yew::{component, html, Html, Properties};

#[derive(Properties, PartialEq)]
pub struct Props {
    pub name: AttrValue,
    #[prop_or_default]
    pub is_loading: bool,
    #[prop_or(AttrValue::Static("default"))]
    pub label: AttrValue,
    #[prop_or_else(|| AttrValue::from("computed"))]
    pub dynamic: AttrValue,
}

#[component]
fn Greeting(&Props { ref name, is_loading, .. }: &Props) -> Html {
    if is_loading { return html! { "Loading" }; }
    html! { <p>{"Hello, "}{name}</p> }
}

// Usage:
html! { <Greeting name="Alice" is_loading=false /> }
```

**Children**: Use `children: Html` field name in props to receive nested content.

```rust
#[derive(Properties, PartialEq)]
pub struct ContainerProps {
    pub children: Html,
}

#[component]
fn Container(props: &ContainerProps) -> Html {
    html! { <div class="wrapper">{ props.children.clone() }</div> }
}
```

**Auto-props** (`yew-autoprops` crate): Generate `Properties` struct from function args.

```rust
use yew_autoprops::autoprops;
// #[autoprops] must appear BEFORE #[component]
#[autoprops]
#[component]
fn Greetings(#[prop_or_default] is_loading: bool, message: &AttrValue) -> Html {
    html! { <>{message}</> }
}
```

### Callbacks

`Callback<IN, OUT>` wraps `Fn` in `Rc`. Use `Callback::from(|e| ...)` for event handlers. Pass down as props for child-to-parent communication.

```rust
// Passing callbacks as props
#[derive(Properties, PartialEq)]
pub struct ButtonProps {
    pub on_click: Callback<Video>,
}

// Creating callbacks with state
let selected = use_state(|| None);
let on_select = {
    let selected = selected.clone();
    Callback::from(move |video: Video| selected.set(Some(video)))
};
```

### Hooks

Rules: (1) name starts with `use_`, (2) call only at top-level of component/hook, (3) same call order every render.

**Pre-defined hooks**:

- `use_state` / `use_state_eq` — reactive local state
- `use_reducer` / `use_reducer_eq` — complex state via reducer pattern
- `use_effect` / `use_effect_with` — side effects (not run during SSR)
- `use_memo` — memoized computation
- `use_callback` — memoized callback
- `use_context` — consume context value
- `use_ref` / `use_mut_ref` — persistent mutable reference
- `use_node_ref` — DOM node reference
- `use_force_update` — manual re-render trigger

```rust
// State
let counter = use_state(|| 0);
counter.set(*counter + 1);

// Effect with deps (runs when deps change)
{
    let data = data.clone();
    use_effect_with(deps, move |_| {
        // setup logic
        || () // cleanup
    });
}

// Reducer
use yew::prelude::*;

enum Action { Increment, Decrement }
struct State { count: i32 }

impl Reducible for State {
    type Action = Action;
    fn reduce(self: Rc<Self>, action: Action) -> Rc<Self> {
        match action {
            Action::Increment => Self { count: self.count + 1 }.into(),
            Action::Decrement => Self { count: self.count - 1 }.into(),
        }
    }
}

let state = use_reducer(|| State { count: 0 });
state.dispatch(Action::Increment);
```

**Custom hooks**: Annotate with `#[hook]`, compose from built-in hooks.

```rust
use yew::prelude::*;

#[hook]
pub fn use_toggle(initial: bool) -> (bool, Callback<()>) {
    let state = use_state(move || initial);
    let toggle = {
        let state = state.clone();
        Callback::from(move |_| state.set(!*state))
    };
    (*state, toggle)
}
```

## html! Macro

For detailed syntax rules, element attributes, conditional rendering, fragments, and lists, see [references/html_and_events.md](references/html_and_events.md).

Key rules:

1. Single root node required (use `<>...</>` fragments for multiple)
2. String literals must be quoted and wrapped in braces: `{ "text" }`
3. Tags must self-close (`<br />`) or have matching close tags
4. Rust expressions go in `{ ... }`
5. `if` / `if let` for conditional rendering
6. `for item in iter { ... }` for lists (add `key` prop)

```rust
html! {
    <>
        <h1>{ "Title" }</h1>
        if show_subtitle {
            <p>{ "Subtitle" }</p>
        }
        for item in &items {
            <li key={item.id}>{ &item.name }</li>
        }
    </>
}
```

## Routing

Add `yew-router` crate. See [references/routing.md](references/routing.md) for nested routers, basename, and query parameters.

```rust
use yew::prelude::*;
use yew_router::prelude::*;

#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[at("/")]
    Home,
    #[at("/post/:id")]
    Post { id: String },
    #[not_found]
    #[at("/404")]
    NotFound,
}

fn switch(route: Route) -> Html {
    match route {
        Route::Home => html! { <h1>{"Home"}</h1> },
        Route::Post { id } => html! { <p>{format!("Post {}", id)}</p> },
        Route::NotFound => html! { <h1>{"404"}</h1> },
    }
}

#[component]
fn App() -> Html {
    html! {
        <BrowserRouter>
            <Switch<Route> render={switch} />
        </BrowserRouter>
    }
}
```

Navigation: `<Link<Route> to={Route::Home}>{"Home"}</Link<Route>>` or `use_navigator()` hook.

## Contexts

Provide shared state without prop drilling. See [references/contexts_and_state.md](references/contexts_and_state.md).

```rust
#[derive(Clone, PartialEq)]
struct Theme { foreground: String, background: String }

// Provider
#[component]
fn App() -> Html {
    let theme = Theme { foreground: "#000".into(), background: "#fff".into() };
    html! {
        <ContextProvider<Theme> context={theme}>
            <ThemedButton />
        </ContextProvider<Theme>>
    }
}

// Consumer
#[component]
fn ThemedButton() -> Html {
    let theme = use_context::<Theme>().expect("no theme context");
    html! { <button style={format!("color: {}", theme.foreground)}>{"Click"}</button> }
}
```

## Data Fetching

Use `gloo-net` for HTTP, `serde` for deserialization, `wasm-bindgen-futures` for async.

```toml
[dependencies]
yew = { version = "0.22", features = ["csr", "serde"] }
gloo-net = "0.6"
serde = { version = "1.0", features = ["derive"] }
wasm-bindgen-futures = "0.4"
```

```rust
use gloo_net::http::Request;
use yew::prelude::*;

#[component]
fn App() -> Html {
    let data = use_state(Vec::new);
    {
        let data = data.clone();
        use_effect_with((), move |_| {
            let data = data.clone();
            wasm_bindgen_futures::spawn_local(async move {
                let resp: Vec<Item> = Request::get("/api/items")
                    .send().await.unwrap()
                    .json().await.unwrap();
                data.set(resp);
            });
            || ()
        });
    }
    // render data...
    html! {}
}
```

## Events and DOM Interaction

See [references/html_and_events.md](references/html_and_events.md) for full event type table and typed event targets.

```rust
// Input handling with TargetCast
use web_sys::HtmlInputElement;
use yew::prelude::*;

#[component]
fn TextInput() -> Html {
    let value = use_state(String::default);
    let oninput = {
        let value = value.clone();
        Callback::from(move |e: InputEvent| {
            let input: HtmlInputElement = e.target_unchecked_into();
            value.set(input.value());
        })
    };
    html! { <input type="text" {oninput} value={(*value).clone()} /> }
}
```

## Server-Side Rendering

See [references/ssr.md](references/ssr.md) for SSR setup, hydration, Suspense integration, and component lifecycle during SSR.

## Agents (Web Workers)

Offload tasks to web workers for concurrent processing. Use `yew-agent` crate. Agents support Public (singleton) and Private (per-bridge) modes. Communication via bridges (bidirectional) and dispatchers (unidirectional). Messages serialized with `bincode`.

## Anti-Patterns to Avoid

1. **`String` in props** — Use `AttrValue` (cheap clone via `Rc<str>`)
2. **`Vec<T>` in props** — Use `IArray<T>` from `implicit-clone`
3. **Interior mutability** (`RefCell`, `Mutex`) — Yew can't detect changes; use hooks
4. **Side effects in render** — Move to `use_effect` / `use_effect_with`
5. **Missing keys on lists** — Always add `key` to iterated elements
6. **Accessing Web APIs during SSR** — Isolate in `use_effect` (not run server-side)

## Ecosystem & Common Dependencies

| Crate                  | Purpose                                                    |
| ---------------------- | ---------------------------------------------------------- |
| `yew-router`           | Client-side routing                                        |
| `gloo`                 | Ergonomic web APIs (net, timers, storage, events, dialogs) |
| `gloo-net`             | HTTP requests (fetch)                                      |
| `wasm-bindgen`         | Rust ↔ JS interop                                          |
| `wasm-bindgen-futures` | Async/await in WASM                                        |
| `web-sys`              | Raw Web API bindings                                       |
| `js-sys`               | JavaScript standard built-in bindings                      |
| `serde` / `serde_json` | Serialization                                              |
| `stylist`              | CSS-in-Rust                                                |
| `yewdux`               | Redux-like state management                                |
| `yew-hooks`            | Community hooks collection                                 |
| `yew-autoprops`        | Auto-generate props structs                                |
| `bounce`               | State management (Redux/Recoil-inspired)                   |
| `implicit-clone`       | Cheap-clone types (`IString`, `IArray`)                    |

## References

- **HTML macro, events, and DOM interaction**: [references/html_and_events.md](references/html_and_events.md)
- **Routing (yew-router) patterns**: [references/routing.md](references/routing.md)
- **Contexts and state management**: [references/contexts_and_state.md](references/contexts_and_state.md)
- **Server-side rendering and hydration**: [references/ssr.md](references/ssr.md)
