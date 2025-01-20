---
title: Typesafe Frontend Routing in Rust/Leptos
date: 2025-01-20
---

Presumed audience background: familiarity with web development, and Rust. 

## TL;DR and take-away

I prototyped a type-safe routing solution for [_leptos_](https://github.com/leptos-rs/leptos/), the full-stack Rust web framework. The approach comprises:

1. a DSL for defining routes parsed with a macro,
2. magic handler functions,
3. a macro to construct typesafe URLs.

If a better alternative for typed routing wasn't available, I personally would think the benefits of type safety outweigh the downsides of the approach (ie, consequences of macro and trait magic). I'm keen to see how [_leptos-routable_](https://github.com/geoffreygarrett/leptos-routable/blob/main/examples/basic-nested-router/src/lib.rs) will shape up, because it also takes aim at type-safe routing in _leptos_. And I suspect it will end up as the better approach.

You can take a look at how it looks in [an actual
codebase here](https://github.com/dnaaun/heimisch/blob/8a4e444146875a46a19fbf7e87908cd520b34813/web/src/app/routing.rs#L19).

## What do I mean by "typesafe routing"?
In a web app, one usually has a bunch of routes (let's use _react_ / _react-router_ as
an accessible example):

```tsx
<Routes>
  <Route path="dashboard" element={<Dashboard />}>
    <Route index element={<Home />} />
    <Route path="teams/:teamId" element={<Team />} />
  </Route>
</Routes>

```

And one links to them elsewhere in the app:

```tsx
<Link to={`/dashboard/teams/${myTeamId}`}>Dashboard</Link>
```

Also, one can access the `teamId` "route param" in the `Team` component by doing something
like:

```tsx
function Team() {
  let teamId = useParams().teamId; // Will have `string | undefined` type.
}
```
As is, the above doesn't provide type safety that prevents:

1. Construction of invalid URLs (when, for example, code constructing URLs becoming stale after restructuring routes).
2. Accessing non-existent route params.

_react-router_ has a solution to this via [code generation](https://reactrouter.com/how-to/route-module-type-safety). [`tanstack/router`](https://tanstack.com/router/latest), a more modern routing solution in TS-land, was built with type-safety front of mind.

I'm working on a web app using the _leptos_ full-stack framework in Rust. And the [builtin routing solution](https://book.leptos.dev/router/16_routes.html) has no type-safety facilities. So I took a shot at prototyping one.

## An explainer on _outlets_/_layouts_
In addition to the two goals of type-safe routing above, I also wanted my prototype to tackle:

3. Type-safe passing of arguments to _outlets_.

Going back to the _react-router_ example from up top, notice how the URL `/dashboard/teams/:teamId` would match both the `<Dashboard />` element specified on line 2 above, and the `<Team />` element specified on line 4? How that plays out is that the `Dashboard` component would have to be defined in the following manner:

```tsx
function Dashboard() {
    return (
        <div>
            {
                // Stuff that's common to everything that gets rendered
                // under /dashboard/ 

            }
            <Outlet />
        </div>
    )
}
```

_react-router_ would then render the `<Team />` element at the spot that `<Outlet />` was invoked in the structure that `Dashboard` returns. `Dashboard` is often referred to as the _layout component_ in this scenario.

What if one wanted to pass an argument to the `Team` component from `Dashboard`? In _react-router,_ one would have to do:

```typescriptreact
function Dashboard() {
    return (
        <div>
            {
                // Stuff that's common to everything that gets rendered
                // under /dashboard/ 
            }
            <Outlet context=someContextValue />
        </div>
    )
}

// And then:
function Team() {
    // The type here would be `unknown | undefined` unless specifies
    // the type like `useOutletContext<OutletContextType>()`. In that
    // case, it'd be up to the user (and not the type-checker) to
    // ensure that `someContextValue` actually has the
    // `OutletContextType` type.
    let outletContext = useOutletContext();
}
```

One can do something similar in _leptos_ via contexts (with a higher degree of type-safety). But for a while, I thought [a bug](https://github.com/leptos-rs/leptos/discussions/3390) meant I couldn't apply a similar solution for my own specific use-case. But it turns out that I was [holding it wrong](https://discord.com/channels/1031524867910148188/1031525141382959175/1327722560137461861). In any case, I ended up working towards the passing of arguments to outlets as a third goal for my routing prototype.

## What does the "type-safe routing for _leptos_" prototype look like?

### Defining routes
One defines routes like so (note: _zwang_ is just the name of the routing solution I came up with):

```rust
zwang_routes! {{
    fallback: NotFound,
    view: Sidebar,
    children: [
        {
            path: "/auth",
            view: Auth
        },
        {
            path: "/{owner_name}",
            layout: Sidebar,
            children: [
                {
                    path: "/{repo_name}",
                    layout: RepositoryPage,
                    view: IssuesList,
                    will_pass: RepositoryId,
                    children: [
                        {
                            path: "/pulls",
                            view: PullRequestsTab
                        },
                        {
                            path: "/issues",
                            will_pass: RepositoryId,
                            view: IssuesList,
                            children: [
                                {
                                    path: "/new",
                                    view: NewIssue
                                },
                                {
                                    path: "/{issue_number}",
                                    view: OneIssue
                                },
                            ]
                        },
                    ]
                }
            ]
        }
    ]
}}
```

### Constructing type-checked URLs

```rust
let owner_name = "google";
let repo_name = "material-design-icons";
zwang_url!("/owner_name={google}/repo_name={repo_name}/issues/issue_number=1")
```

### Type-safe access to route params

```rust
fn IssuesList(
    params: RouteParams<ParamsOwnerNameRepoName>,
) -> impl IntoView {
 // ...
} 
```

The `ParamsOwnerNameRepoName` is a struct created by the `zwang_routes!` macro. The rule
of how the name is constructed is that the name of every route param available is pascal-cased, alphabetized, and
then concatenated (your IDE will helpfully show it in autocomplete).

If, for instance, we tried to access the `issue_number` route param at `IssueList`
(which doesn't make sense since none of the routes that `IssueList` gets rendered at
include such a param, we'd get a compiler error):

```rust
fn IssuesList(
    params: RouteParams<ParamsIssueNumberOwnerNameRepoName>,
) -> impl IntoView {
 // ...
} 
```

### Passing arguments to _outlets_

See the lines that say `will_pass: RepositoryId` in the route specification
above? They indicate to the `zwang_routes!` macro that the component will pass an
argument of said type. Therefore, the _layout_ component can take an argument like:

```rust
pub fn RepositoryPage(
    outlet: Outlet<RepositoryId, impl IntoView + 'static>,
) -> impl IntoView {
    let repository_id = get_repository_id(); // or something.
    view! { // the leptos macro for "JSX-in-rust"
        <div>
            {outlet.call(repository_id)}
        </div>
    }
}
```

And then the _outlet_ component can receive that argument as follows:

```rust
pub fn IssuesList(repository_id: ArgFromLayout<RepositoryId>) -> impl IntoView {
  // ...
}
```

We make use of [_axum_-style "magic handler functions"](https://lunatic.solutions/blog/magic-handler-functions-in-rust/), and so a component that we hook up in our routes can decide to take as few or as many of the "magic arguments" that we talked about above. For example:

```rust
pub fn IssuesList(
    params: RouteParams<ParamsOwnerNameRepoName>,
    repository_id: ArgFromLayout<Signal<RepositoryId>>,
) -> impl IntoView {
    // ...
}
```

## Fundamental shortcomings of approach
Given the approach I outlined above, I think the following are fundamental limitations unlikely to be improved with further work: 

1. **Macro magic**: In general, I'm given to like complex macros like `zwang_routes!` only if I wrote them in the first place, because it's often only then that I transparently understand how they work. 
2. **Trait magic**: The _axum_-style magic functions work with trait magic. And debugging why trait-level logic is rejecting whatever you're trying to pass it is challenging enough that _axum_ came up with a [macro annotation](https://docs.rs/axum/latest/axum/attr.debug_handler.html) to help with it.


