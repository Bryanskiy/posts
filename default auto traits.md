# Default auto traits

This text is a description of [#120706](https://github.com/rust-lang/rust/pull/120706) PR which is part of ["MCP: Low level components for async drop" ](https://github.com/rust-lang/compiler-team/issues/727)

## Intro

Sometimes we want to use type system to express specific behavior and provide safety guarantees. This behavior can be specified by various "marker" traits. For example, we use `Send` and `Sync` to keep track of which types are thread safe. As the language develops, there are more problems that could be solved by adding new marker traits:

- to forbid types with an async destructor to be dropped in a synchronous context a trait like `SyncDrop` could be used [Async destructors, async genericity and completion futures](https://sabrinajewson.org/blog/async-drop).
-  to support [scoped tasks](https://without.boats/blog/the-scoped-task-trilemma/) or in a more general sense to provide a [destruction guarantee](https://zetanumbers.github.io/book/myosotis.html)  there is a desire among some users to see a `Leak` (or `Forget`) trait.
- Withoutboats in his [post](https://without.boats/blog/changing-the-rules-of-rust/)  reflected on the use of `Move` trait instead of a `Pin`.

All the traits proposed above are supposed to be auto traits implemented for most types, and usually implemented automatically by compiler.

For backward compatibility these traits have to be added implicitly to all bound lists in old code (see below).
Adding new default bounds involves many difficulties: many standard library interfaces may need to opt out of those default bounds, and therefore be infected with confusing `?Trait` syntax, migration to a new edition may contain backward compatibility holes, supporting new traits in the compiler can be quite difficult and so forth. Anyway, it's hard to evaluate the complexity until we try the system on a practice.

In this PR we introduce new optional lang items for traits that are added to all bound lists by default, similarly to existing `Sized`. The examples of such traits could be `Leak`, `Move`, `SyncDrop` or something else, it doesn't matter much right now (further I will call them `DefaultAutoTrait`'s).  We want to land this change into rustc under an option, so it becomes available in bootstrap compiler. Then we'll be able to do standard library experiments with the aforementioned traits without adding hundreds of `#[cfg(not(bootstrap))]`s. Based on the experiments, we can come up with some scheme for the next edition, in which such bounds are added in a more targeted way, and not just everywhere.

Most of the implementation is basically a refactoring that replaces hardcoded uses of `Sized` with iterating over a list of traits including both `Sized` and the new traits when `-Zexperimental-default-bounds` is enabled (or just `Sized` as before, if the option is not enabled).

## Default bounds for old editions

All existing types, including generic parameters, are considered `Leak`/`Move`/`SyncDrop` and can be forgotten, moved or destroyed in generic contexts without specifying any bounds.  New types that cannot be, for example, forgotten and do not implement `Leak` can be added at some point, and they should not be usable in such generic contexts in existing code.

To both maintain this property and keep backward compatibility with existing code, the new traits should be added as default bounds _everywhere_ in previous editions. Besides the implicit `Sized` bound contexts that includes supertrait lists and trait lists in trait objects (`dyn Trait1 + ... + TraitN`). Compiler should also generate implicit `DefaultAutoTrait` implementations for foreign types (`extern { type Foo; }`) because they are also currently usable in generic contexts without any bounds.

### Supertraits

Adding the new traits as supertraits to all existing traits is potentially necessary, because, for example, using a `Self` param in a trait's associated item may be a breaking change otherwise:

```rust
trait Foo: Sized {
    fn new() -> Option<Self>; // ERROR: `Option` requires `DefaultAutoTrait`, but `Self` is not `DefaultAutoTrait`
}

// desugared `Option`
enum Option<T: DefaultAutoTrait + Sized> {
    Some(T),
    None,
}
```

However, default supertraits can significantly affect compiler performance. For example, if we know that `T: Trait`, the compiler would deduce that `T: DefaultAutoTrait`. It also implies proving `F: DefaultAutoTrait` for each field `F`  of type `T` until an explicit impl is be provided.

If the standard library is not modified, then even traits like `Copy` or `Send` would get these supertraits.

In this PR for optimization purposes instead of adding default supertraits, bounds are added to the associated items:

```rust
// Default bounds are generated in the following way:
trait Trait {
   fn foo(&self) where Self: DefaultAutoTrait {}
}

// instead of this:
trait Trait: DefaultAutoTrait {
   fn foo(&self) {}
}
```

It is not always possible to do this optimization because of backward compatibility:
```rust
pub trait Trait<Rhs = Self> {}
pub trait Trait1 : Trait {} // ERROR: `Rhs` requires `DefaultAutoTrait`, but `Self` is not `DefaultAutoTrait`
```

or

```rust
trait Trait {
   type Type where Self: Sized;
}
trait Trait2<T> : Trait<Type = T> {} // ERROR: `???` requires `DefaultAutoTrait`, but `Self` is not `DefaultAutoTrait`
```

Therefore, `DefaultAutoTrait`'s are still being added to supertraits if the `Self` params or type bindings were found in the trait header.

### Trait objects

Trait objects requires explicit `+ Trait` bound to implement  corresponding trait which is not backward compatible:

```rust
fn use_trait_object(x: Box<dyn Trait>) {
   foo(x) // ERROR: `foo` requires `DefaultAutoTrait`, but `dyn Trait` is not `DefaultAutoTrait`
}

// implicit T: DefaultAutoTrait here
fn foo<T>(_: T) {}
```

So, for a trait object `dyn Trait` we should add an implicit bound `dyn Trait + DefaultAutoTrait` to make it usable, and allow relaxing it with a question mark syntax `dyn Trait + ?DefaultAutoTrait` when it's not necessary.

### Foreign types

If compiler doesn't generate auto trait implementations for a foreign type, then it's a breaking change if the default bounds are added everywhere else:

```rust
// implicit T: DefaultAutoTrait here
fn foo<T: ?Sized>(_: &T) {}

extern "C" {
    type ExternTy;
}

fn forward_extern_ty(x: &ExternTy) {
    foo(x); // ERROR: `foo` requires `DefaultAutoTrait`, but `ExternTy` is not `DefaultAutoTrait`
}
```

We'll have to enable implicit `DefaultAutoTrait` implementations for foreign types at least for previous editions:

```rust
// implicit T: DefaultAutoTrait here
fn foo<T: ?Sized>(_: &T) {}

extern "C" {
    type ExternTy;
}

impl DefaultAutoTrait for ExternTy {} // implicit impl

fn forward_extern_ty(x: &ExternTy) {
    foo(x); // OK
}
```

## Unresolved questions

New default bounds affect all existing Rust code complicating an already complex type system.

- Proving an auto trait predicate requires recursively traversing the type and proving the predicate for it's fields. This leads to a significant performance regression. Measurements for the stage 2 compiler build show up to 3x regression.
  - We hope that fast path optimizations for well known traits could mitigate such regressions at least partially.
- New default bounds trigger some compiler bugs in both old and new trait solver.
- With new default bounds we encounter some trait solver cycle errors that break existing code.
  - We hope that these cases are bugs that can be addressed in the new trait solver.

Also migration to a new edition could be quite ugly and enormous, but that's actually what we want to solve. For other issues there's a chance that they could be solved by a new solver.

# Literature

Most of the content in this explainer is based on the following posts:

- [https://blog.yoshuawuyts.com/linear-types-one-pager/](https://blog.yoshuawuyts.com/linear-types-one-pager/)
- [https://without.boats/blog/changing-the-rules-of-rust/](https://without.boats/blog/changing-the-rules-of-rust/)
- [https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/)
- https://sabrinajewson.org/blog/async-drop

But the literature from MPC is also relevant.
