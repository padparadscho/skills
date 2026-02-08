# HTML Macro, Events & DOM Interaction

## Table of Contents

- [html! Macro Syntax](#html-macro-syntax)
- [Fragments](#fragments)
- [Elements and Attributes](#elements-and-attributes)
- [Conditional Rendering](#conditional-rendering)
- [Lists and Iteration](#lists-and-iteration)
- [Keyed Lists](#keyed-lists)
- [Event Handling](#event-handling)
- [Event Types Table](#event-types-table)
- [Typed Event Targets](#typed-event-targets)
- [NodeRef for DOM Access](#noderef-for-dom-access)
- [Manual Event Listeners](#manual-event-listeners)

## html! Macro Syntax

The `html!` macro produces `Html` (alias for `VNode`). Core rules:

1. **Single root** — wrap multiple elements in `<>...</>` (fragment)
2. **Literals in braces** — `{ "Hello" }`, not bare `Hello`
3. **Expressions in braces** — `{ variable }`, `{ format!("x: {}", val) }`
4. **Tags must close** — `<div></div>` or self-closing `<input />`
5. **Recursion limit** — add `#![recursion_limit="1024"]` at crate root if needed

```rust
use yew::prelude::*;

let header = "Welcome";
let count: usize = 42;

html! {
    <div>
        <h1>{ header }</h1>
        <p>{ "Count: " }{ count }</p>
    </div>
}
```

## Fragments

Empty tags `<>...</>` group elements without adding a DOM node:

```rust
html! {
    <>
        <h1>{ "Title" }</h1>
        <p>{ "Paragraph" }</p>
    </>
}
```

An empty `html! {}` renders nothing.

## Elements and Attributes

Set attributes like HTML. Braces optional for literals:

```rust
html! {
    <div>
        <div data-key="abc"></div>
        <input type="text" id="name" value="placeholder" />
        <input type="checkbox" checked=true />
        <textarea value="content" />
        <select name="choice">
            <option selected=true value="">{ "Selected" }</option>
            <option selected=false disabled=true value="">{ "Disabled" }</option>
        </select>
    </div>
}
```

### Dynamic Tag Names

Required for SVG elements with uppercase characters:

```rust
html! { <@{"myTag"}></@> }
```

### Special Properties

- `ref` — attach a `NodeRef` for DOM access
- `key` — unique identifier for list optimization

### Attribute Properties (setting DOM attributes vs properties)

```rust
// Standard attributes
html! { <div attribute={value} /> }

// DOM properties (rare, use ~ prefix)
html! { <my-element ~property="abc" /> }
```

## Conditional Rendering

Use `if` and `if let` directly in macro body:

```rust
html! {
    <>
        if show_header {
            <h1>{ "Header" }</h1>
        }
        if let Some(user) = &*current_user {
            <p>{ format!("Hello, {}", user.name) }</p>
        }
    </>
}
```

## Lists and Iteration

Three approaches:

### for loops (recommended)

```rust
html! {
    for item in &items {
        <li key={item.id}>{ &item.name }</li>
    }
}
```

### for blocks

```rust
html! {
    { for items.iter().map(|item| html! { <li key={item.id}>{ &item.name }</li> }) }
}
```

### collect method

```rust
let items_html: Vec<Html> = items.iter()
    .map(|item| html! { <li key={item.id}>{ &item.name }</li> })
    .collect();
html! { <ul>{ items_html }</ul> }
```

## Keyed Lists

Always add `key` prop to iterated elements. Keys must be unique within the list (not globally). Keys enable Yew to efficiently reconcile list changes (reorder, insert, delete) without recreating DOM nodes.

```rust
html! {
    <div>
        { names.into_iter().map(|name| {
            html! { <div key={name}>{ format!("Hello, {}!", name) }</div> }
        }).collect::<Html>() }
    </div>
}
```

Performance: keyed lists can be ~2x faster for reordering operations compared to unkeyed lists.

## Event Handling

Attach event callbacks with `on*` attributes. Events use `web-sys` types re-exported under `yew::events`.

```rust
use yew::prelude::*;

#[component]
fn Button() -> Html {
    let onclick = Callback::from(|e: MouseEvent| {
        // handle click
    });
    html! { <button {onclick}>{ "Click" }</button> }
}
```

Event shorthand — when variable name matches attribute name:

```rust
let onclick = Callback::from(|_| { /* ... */ });
html! { <button {onclick}>{ "Click" }</button> }
// equivalent to: <button onclick={onclick}>
```

### Event Bubbling

Events bubble up through the virtual DOM hierarchy. Disable globally for performance if not needed:

```rust
yew::set_event_bubbling(false); // before app start
```

### Event Delegation

Yew delegates event listeners to the app subtree root (not individual elements). Consequences:

- `Event::current_target` points to subtree root, not the element
- `Event::event_phase` is always `CAPTURING_PHASE`
- Yew listeners fire before other JS listeners

## Event Types Table

| Listener                                                     | Event Type        |
| ------------------------------------------------------------ | ----------------- |
| `onclick`                                                    | `MouseEvent`      |
| `ondblclick`                                                 | `MouseEvent`      |
| `oncontextmenu`                                              | `MouseEvent`      |
| `onmousedown/up/enter/leave/move/over/out`                   | `MouseEvent`      |
| `onkeydown/keyup/keypress`                                   | `KeyboardEvent`   |
| `oninput`                                                    | `InputEvent`      |
| `onchange`                                                   | `Event`           |
| `onsubmit`                                                   | `SubmitEvent`     |
| `onfocus/onblur`                                             | `FocusEvent`      |
| `onfocusin/onfocusout`                                       | `FocusEvent`      |
| `ondrag/dragstart/dragend/dragenter/dragleave/dragover/drop` | `DragEvent`       |
| `onscroll/onresize`                                          | `Event`           |
| `onwheel`                                                    | `WheelEvent`      |
| `ontouchstart/touchend/touchmove/touchcancel`                | `TouchEvent`      |
| `onpointerdown/up/enter/leave/move/over/out/cancel`          | `PointerEvent`    |
| `onanimationstart/end/iteration/cancel`                      | `AnimationEvent`  |
| `ontransitionrun/start/end/cancel`                           | `TransitionEvent` |
| `onload/onerror`                                             | `Event`           |
| `onloadstart/onprogress/onloadend`                           | `ProgressEvent`   |
| `oncopy/oncut/onpaste`                                       | `Event`           |

## Typed Event Targets

### Using TargetCast (recommended)

Yew's `TargetCast` trait provides `target_dyn_into` (safe) and `target_unchecked_into` (unsafe):

```rust
use web_sys::HtmlInputElement;
use yew::prelude::*;

let on_change = Callback::from(move |e: Event| {
    // Safe cast
    if let Some(input) = e.target_dyn_into::<HtmlInputElement>() {
        let value = input.value();
    }
});

// Or unsafe (when certain of element type)
let on_change = Callback::from(move |e: Event| {
    let value = e.target_unchecked_into::<HtmlInputElement>().value();
});
```

### Using JsCast

From `wasm-bindgen::JsCast`:

```rust
use wasm_bindgen::JsCast;
use web_sys::{EventTarget, HtmlInputElement};

let on_change = Callback::from(move |e: Event| {
    let target: Option<EventTarget> = e.target();
    let input = target.and_then(|t| t.dyn_into::<HtmlInputElement>().ok());
    if let Some(input) = input {
        // use input.value()
    }
});
```

## NodeRef for DOM Access

Obtain a reference to the underlying DOM element:

```rust
use web_sys::HtmlInputElement;
use yew::prelude::*;

#[component]
fn MyInput() -> Html {
    let input_ref = use_node_ref();

    let onchange = {
        let input_ref = input_ref.clone();
        Callback::from(move |_| {
            if let Some(input) = input_ref.cast::<HtmlInputElement>() {
                let value = input.value();
                // use value
            }
        })
    };

    html! { <input ref={input_ref} {onchange} type="text" /> }
}
```

> **Caution**: Do not manually modify the DOM tree rendered by Yew. Treat `NodeRef` as read-only access.

## Manual Event Listeners

For events not supported by `html!` macro, use `gloo::events::EventListener`:

```rust
use web_sys::HtmlElement;
use yew::prelude::*;
use gloo::events::EventListener;

#[component]
fn CustomEventComponent() -> Html {
    let div_ref = use_node_ref();

    use_effect_with(div_ref.clone(), {
        let div_ref = div_ref.clone();
        move |_| {
            let mut listener = None;
            if let Some(element) = div_ref.cast::<HtmlElement>() {
                let handler = Callback::from(move |_: Event| { /* handle */ });
                listener = Some(EventListener::new(
                    &element, "customevent",
                    move |e| handler.emit(e.clone())
                ));
            }
            move || drop(listener) // cleanup on unmount
        }
    });

    html! { <div ref={div_ref}></div> }
}
```

Add `gloo-events = "0.1"` to dependencies.

## Lints

With nightly Rust, `html!` provides accessibility-related warnings. Run `cargo +nightly check` to see them even when using stable for builds.
