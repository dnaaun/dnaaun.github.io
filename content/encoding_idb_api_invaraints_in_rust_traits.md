---
title: Encoding IndexedDB API's Invariants in Rust Traits
date: 2025-01-20
---

Say you're working on [a full-stack Rust web app](https://github.com/dnaaun/heimisch) with [leptos](https://github.com/leptos-rs/leptos), and you want to use the browser's [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API). You could use The [bindings provided by web-sys](https://docs.rs/web-sys/latest/web_sys/struct.Window.html#method.indexed_db) directly, but you come across the [`idb`](https://docs.rs/idb/latest/idb/) crate, which exposes a nicer to use futures-based API (as opposed to a callback-based API).

```rust
fn some_func() -> bool {
}
```
