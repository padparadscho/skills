# Routing with yew-router

## Table of Contents

- [Setup](#setup)
- [Defining Routes](#defining-routes)
- [Switch and BrowserRouter](#switch-and-browserrouter)
- [Path Segments and Parameters](#path-segments-and-parameters)
- [Navigation](#navigation)
- [Query Parameters](#query-parameters)
- [Nested Routers](#nested-routers)
- [Basename](#basename)
- [Listening to Route Changes](#listening-to-route-changes)

## Setup

```toml
[dependencies]
yew-router = "0.18"
# or
# cargo add yew-router
```

Import: `use yew_router::prelude::*;`

## Defining Routes

Derive `Routable` on an enum. Each variant maps to a URL path pattern.

```rust
use yew_router::prelude::*;

#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[at("/")]
    Home,
    #[at("/about")]
    About,
    #[at("/post/:id")]
    Post { id: String },
    #[at("/*path")]
    Misc { path: String },
    #[not_found]
    #[at("/404")]
    NotFound,
}
```

- Fields must implement `Clone + PartialEq + Display + FromStr`
- Primitive types (`String`, `u32`, etc.) satisfy this automatically
- `#[not_found]` designates the fallback route
- If deserialization of a segment fails (e.g., `u8` overflow), the route is treated as unmatched

## Switch and BrowserRouter

`<Switch>` renders the matched route. Must be inside a `<BrowserRouter>`.

```rust
use yew::prelude::*;
use yew_router::prelude::*;

fn switch(route: Route) -> Html {
    match route {
        Route::Home => html! { <h1>{"Home"}</h1> },
        Route::About => html! { <About /> },
        Route::Post { id } => html! { <PostPage {id} /> },
        Route::NotFound => html! { <h1>{"404 Not Found"}</h1> },
        _ => html! {},
    }
}

#[component]
fn App() -> Html {
    html! {
        <BrowserRouter>
            <nav>
                <Link<Route> to={Route::Home}>{"Home"}</Link<Route>>
                {" | "}
                <Link<Route> to={Route::About}>{"About"}</Link<Route>>
            </nav>
            <Switch<Route> render={switch} />
        </BrowserRouter>
    }
}
```

## Path Segments and Parameters

Dynamic segments use `:name` syntax. Named wildcards use `*name`.

```rust
#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[at("/user/:user_id/post/:post_id")]
    UserPost { user_id: u32, post_id: u32 },
    #[at("/files/*path")]
    Files { path: String },
}
```

When segment deserialization fails (e.g., non-numeric value for `u32`), the router falls through to `#[not_found]`.

## Navigation

### Link Component

Renders as `<a>` with SPA behavior (no page reload):

```rust
<Link<Route> to={Route::Home}>{"Go Home"}</Link<Route>>
<Link<Route> to={Route::Post { id: "hello".into() }}>{"Read Post"}</Link<Route>>
```

### Navigator API (programmatic)

Use `use_navigator()` hook in function components:

```rust
#[component]
fn MyComponent() -> Html {
    let navigator = use_navigator().unwrap();

    let go_home = {
        let navigator = navigator.clone();
        Callback::from(move |_| navigator.push(&Route::Home))
    };

    let go_post = {
        let navigator = navigator.clone();
        Callback::from(move |_| navigator.push(&Route::Post { id: "42".into() }))
    };

    html! {
        <>
            <button onclick={go_home}>{"Home"}</button>
            <button onclick={go_post}>{"Post 42"}</button>
        </>
    }
}
```

- `navigator.push(&route)` — push new history entry
- `navigator.replace(&route)` — replace current history entry
- `navigator.push_with_query(&route, &query)` — push with query params
- `navigator.replace_with_query(&route, &query)` — replace with query params

> **Caution**: Pushing the same route twice can panic. Use `Callback::from` only when the route differs from current; otherwise use standard `Callback`.

For struct components, use `ctx.link().navigator().unwrap()`.

### Redirect Component

```rust
#[component]
fn ProtectedPage() -> Html {
    let user = use_user(); // hypothetical hook
    match user {
        Some(user) => html! { <Dashboard {user} /> },
        None => html! { <Redirect<Route> to={Route::Login} /> },
    }
}
```

Use `<Redirect>` when redirecting from render logic. Use `Navigator` API for redirecting from callbacks.

## Query Parameters

### Setting query parameters

Serialize with `serde`:

```rust
use serde::Serialize;

#[derive(Serialize)]
struct Query {
    page: u32,
    search: String,
}

let navigator = use_navigator().unwrap();
let query = Query { page: 1, search: "rust".into() };
navigator.push_with_query(&Route::Search, &query).unwrap();
```

### Reading query parameters

Deserialize from URL:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Query {
    page: u32,
    search: String,
}

let location = use_location().unwrap();
let query: Query = location.query().unwrap();
```

## Nested Routers

Split large applications into sub-routers:

```rust
#[derive(Clone, Routable, PartialEq)]
enum MainRoute {
    #[at("/")]
    Home,
    #[at("/settings")]
    SettingsRoot,
    #[at("/settings/*")]
    Settings,
    #[not_found]
    #[at("/404")]
    NotFound,
}

#[derive(Clone, Routable, PartialEq)]
enum SettingsRoute {
    #[at("/settings")]
    Profile,
    #[at("/settings/friends")]
    Friends,
    #[at("/settings/theme")]
    Theme,
    #[not_found]
    #[at("/settings/404")]
    NotFound,
}

fn switch_main(route: MainRoute) -> Html {
    match route {
        MainRoute::Home => html! { <h1>{"Home"}</h1> },
        MainRoute::SettingsRoot | MainRoute::Settings => {
            html! { <Switch<SettingsRoute> render={switch_settings} /> }
        }
        MainRoute::NotFound => html! { <h1>{"Not Found"}</h1> },
    }
}

fn switch_settings(route: SettingsRoute) -> Html {
    match route {
        SettingsRoute::Profile => html! { <ProfilePage /> },
        SettingsRoute::Friends => html! { <FriendsPage /> },
        SettingsRoute::Theme => html! { <ThemePage /> },
        SettingsRoute::NotFound => {
            html! { <Redirect<MainRoute> to={MainRoute::NotFound} /> }
        }
    }
}

#[component]
fn App() -> Html {
    html! {
        <BrowserRouter>
            <Switch<MainRoute> render={switch_main} />
        </BrowserRouter>
    }
}
```

## Basename

Set a common prefix for all routes. Either via prop or `<base>` HTML tag.

```rust
html! {
    <BrowserRouter basename="/app">
        <Switch<Route> render={switch} />
    </BrowserRouter>
}
```

Or in `index.html`:

```html
<head>
  <base href="/app" />
</head>
```

Falls back to `/` if neither is set.

## Listening to Route Changes

### Function Components

Use `use_location()` and `use_route::<R>()` hooks. Components re-render when values change.

```rust
#[component]
fn LocationDisplay() -> Html {
    let location = use_location().unwrap();
    let route = use_route::<Route>().unwrap();
    html! { <p>{ format!("Current path: {}", location.path()) }</p> }
}
```

### Struct Components

Register a location listener:

```rust
fn create(ctx: &Context<Self>) -> Self {
    let listener = ctx.link()
        .add_location_listener(ctx.link().callback(|_| Msg::LocationChanged))
        .unwrap();
    Self { _listener: listener } // store to prevent drop
}
```
