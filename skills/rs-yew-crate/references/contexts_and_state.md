# Contexts and State Management

## Table of Contents

- [Context API](#context-api)
- [Providing Context](#providing-context)
- [Consuming Context](#consuming-context)
- [Mutable Contexts with Reducers](#mutable-contexts-with-reducers)
- [State Management Patterns](#state-management-patterns)
- [Yewdux](#yewdux)
- [Bounce](#bounce)
- [When to Use What](#when-to-use-what)

## Context API

Contexts solve prop drilling — passing data through intermediate components that don't need it. A parent provides context; any descendant consumes it without explicit prop passing.

Context types must implement `Clone + PartialEq`.

## Providing Context

Wrap descendants with `ContextProvider<T>`:

```rust
use yew::prelude::*;

#[derive(Clone, Debug, PartialEq)]
struct Theme {
    foreground: String,
    background: String,
}

#[component]
fn App() -> Html {
    let theme = use_state(|| Theme {
        foreground: "#000000".to_owned(),
        background: "#eeeeee".to_owned(),
    });

    html! {
        <ContextProvider<Theme> context={(*theme).clone()}>
            <Toolbar />
        </ContextProvider<Theme>>
    }
}
```

All children and their descendants have access to this context.

## Consuming Context

### Function Components

Use `use_context::<T>()` hook. Returns `Option<T>`.

```rust
#[component]
fn ThemedButton() -> Html {
    let theme = use_context::<Theme>().expect("no theme context found");

    html! {
        <button style={format!("background: {}; color: {};", theme.background, theme.foreground)}>
            { "Click me!" }
        </button>
    }
}
```

Components re-render when context value changes.

### Struct Components

Two approaches:

1. **Higher Order Components (HOC)**: wrap struct component in a function component that consumes context and passes as props
2. **Direct consumption**: see `examples/contexts/src/struct_component_subscriber.rs` in the Yew repo

## Mutable Contexts with Reducers

Rust ownership rules prevent `&mut self` on shared context. Combine context with `use_reducer`:

```rust
use std::rc::Rc;
use yew::prelude::*;

#[derive(Clone, Debug, PartialEq)]
struct AppState {
    user: Option<String>,
    theme: String,
}

enum AppAction {
    Login(String),
    Logout,
    SetTheme(String),
}

impl Reducible for AppState {
    type Action = AppAction;

    fn reduce(self: Rc<Self>, action: AppAction) -> Rc<Self> {
        match action {
            AppAction::Login(name) => Rc::new(Self { user: Some(name), ..(*self).clone() }),
            AppAction::Logout => Rc::new(Self { user: None, ..(*self).clone() }),
            AppAction::SetTheme(theme) => Rc::new(Self { theme, ..(*self).clone() }),
        }
    }
}

#[component]
fn App() -> Html {
    let state = use_reducer(|| AppState {
        user: None,
        theme: "light".to_owned(),
    });

    html! {
        <ContextProvider<UseReducerHandle<AppState>> context={state}>
            <MainContent />
        </ContextProvider<UseReducerHandle<AppState>>>
    }
}

#[component]
fn LoginButton() -> Html {
    let state = use_context::<UseReducerHandle<AppState>>().unwrap();
    let onclick = {
        let state = state.clone();
        Callback::from(move |_| state.dispatch(AppAction::Login("Alice".into())))
    };
    html! { <button {onclick}>{ "Login" }</button> }
}
```

## State Management Patterns

### Local State — `use_state`

For component-scoped state. Cheapest and simplest.

```rust
let count = use_state(|| 0);
let onclick = {
    let count = count.clone();
    Callback::from(move |_| count.set(*count + 1))
};
```

### Shared State — Context + Reducer

For state shared across a subtree. Good for medium-sized apps.

### Complex Shared State — `use_reducer`

For state with many transitions. Define an action enum and implement `Reducible`.

### Memoization — `use_memo`

Cache expensive computations:

```rust
let filtered = use_memo(items.clone(), |items| {
    items.iter().filter(|i| i.active).cloned().collect::<Vec<_>>()
});
```

### Callback Memoization — `use_callback`

Prevent child re-renders from callback identity changes:

```rust
let on_click = use_callback(count.clone(), |_, count| {
    // use count
});
```

## Yewdux

Redux-like global state for Yew. Add `yewdux` crate.

```rust
use yewdux::prelude::*;

#[derive(Default, Clone, PartialEq, Store)]
struct CounterState {
    count: u32,
}

#[component]
fn Counter() -> Html {
    let (state, dispatch) = use_store::<CounterState>();
    let onclick = dispatch.reduce_mut_callback(|state| state.count += 1);

    html! {
        <button {onclick}>{ state.count }</button>
    }
}
```

Key features:

- Global store accessible from any component
- `#[derive(Store)]` macro
- `use_store` hook for read + dispatch
- Selector support for partial subscriptions
- Persistence support

## Bounce

Uncomplicated state management inspired by Redux + Recoil. Add `bounce` crate.

Key concepts:

- **Atoms**: independent pieces of state
- **Slices**: reducible state (like Redux slices)
- **Selectors**: derived state

```rust
use bounce::prelude::*;

#[derive(Atom, PartialEq, Default)]
struct Username {
    value: String,
}

#[component]
fn App() -> Html {
    html! {
        <BounceRoot>
            <UserDisplay />
        </BounceRoot>
    }
}

#[component]
fn UserDisplay() -> Html {
    let username = use_atom::<Username>();
    html! { <p>{ &username.value }</p> }
}
```

## When to Use What

| Scenario                         | Solution                                        |
| -------------------------------- | ----------------------------------------------- |
| Single component state           | `use_state`                                     |
| Complex single component state   | `use_reducer`                                   |
| Share state with direct children | Props                                           |
| Share state across subtree       | Context + `use_reducer`                         |
| Global app state                 | Yewdux or Bounce                                |
| Derived/computed state           | `use_memo` or Bounce selectors                  |
| Form state                       | `use_state` per field or `use_reducer` for form |

### Considerations Before Using Context

- Extract components and pass as children instead of drilling props
- Context causes all consumers to re-render on change — keep contexts focused
- Split large contexts into smaller, domain-specific ones
