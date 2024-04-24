# Default auto traits

This text is a description of [#120706](https://github.com/rust-lang/rust/pull/120706) PR which is part of ["MCP: Low level components for async drop" ](https://github.com/rust-lang/compiler-team/issues/727)

## Intro

Sometimes in Rust, we want to use the type system to express specific behavior and provide safety guarantees. This behavior is specified by certain "marker" traits. For example, we use `Send` and `Sync` to keep track of which types are thread safe. As the language develops, there are more and more problems that could be solved by adding new marker traits:

- to forbid types with an async destructor to be dropped in a synchronous context a certain trait `SyncDrop` could be used [Async destructors, async genericity and completion futures](https://sabrinajewson.org/blog/async-drop).
-  to support [scoped tasks](https://without.boats/blog/the-scoped-task-trilemma/) or in a more general sense to provide a [destruction guarantee](https://zetanumbers.github.io/book/myosotis.html)  there is a desire among some users to see a `Leak` trait.
- Withoutboats in his [post](https://without.boats/blog/changing-the-rules-of-rust/)  reflected on the use of `Move` trait instead of a `Pin`.

Proposed traits semantically are close to auto traits with default bounds.  However, adding new default bounds paired with lots of difficulties: many standard library interfaces may need to be infected with confusing `?Trait` syntax, migration to a new edition may contain backward compatibility holes, supporting new traits in the compiler can be quite difficult and so forth. Anyway, it's difficult to evaluate the complexity until we try the system on a practice.

 In this PR we introduce new lang item traits that are added to all bounds by default, similarly to existing `Sized`. The examples of such traits could be `Leak`, `Move`, `SyncDrop` or something else, it doesn't matter much right now(further i will call them `DefaultAutoTrait`'s).  We want to land the change into rustc under an option, so it becomes available in bootstrap compiler. Then we'll be able to do standard library experiments with the aforementioned traits without adding hundreds of `#[cfg(not(bootstrap))]`s. Based on the experiments, we can come up with some scheme for the next edition, in which such bounds are added with the minimum troubles.
## Default bounds for old editions

To maintain the backward compatibility, default bounds should be added _everywhere_ in previous editions. Besides the `Sized` bound contexts it includes supertraits and trait objects. Compiler also should be able generate `DefaultAutoTrait` implementation for foreign types.
### supertraits

Using a `Self` param in trait associative items could be a breaking change if the trait didn’t imply corresponding bounds as a supertrait:

```rust
trait Foo: Sized {
    fn new() -> Option<Self>; // ERROR: `Self: DefaultAutoTrait` is not satisfied
}


// desugared `Option`
enum Option<T: DefaultAutoTrait + Sized> {
    Some(T),
    None,
}
```

However, default supertraits can significantly affect compiler performance. For example, if we know that `T: Trait`, the compiler would deduce that `T: DefaultAutoTrait`. It also implies proving `F: DefaultAutoTrait` for each field `F`  of type `T` until an explicit impl will be provided.

In this PR for optimization purposes instead of adding the default supertraits, bounds are added to the associative items:

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
pub trait Trait1 : Trait {} // ERROR
```

or

```rust
trait Trait {
   type Type where Self: Sized;
}
trait Trait2<T> : Trait<Type = T> {} // ERROR
```

Therefore, `DefaultAutoTrait`'s are still being added to supertraits if the `Self` params or type bindings were found in the trait header.
### trait objects

Trait objects requires explicit `+ Trait` bound to implement  corresponding trait which is not backward compatible:

```rust
fn use_trait_object(x: Box<dyn Trait>) {
   foo(x) // ERROR
}

// implicit T: DefaultAutoTrait here
fn foo<T>(_: T) {}
```

So, for a trait object `dyn Trait` we should add an implicit bound `dyn Trait + DefaultAutoTrait` and allow relaxing it with a question mark syntax `dyn Trait + ?DefaultAutoTrait` to make it usable.
### foreign types

Compiler couldn't generate auto trait implementations for foreign types since their content is unknown which is breaking change:

```rust
// implicit T: DefaultAutoTrait here
fn foo<T: ?Sized>(_: &T) {}

extern "C" {
    type ExternTy;
}

fn forward_extern_ty(x: &ExternTy) {
    foo(x); // ERROR
}
```

We're going to have to allow an implicit `DefaultAutoTrait` implementations for foreign types at least for previous editions:

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
- New default bounds trigger some compiler bugs.
- We encounter some cycle errors that can break an existing code.

 Also migration to a new edition could be quite ugly and enormous, but that's actually what we a want to solve. For other issues there's a chance that they could be solved by a new solver.

# Literature

Most of the content in this explainer is based on the following posts:

- [https://blog.yoshuawuyts.com/linear-types-one-pager/](https://blog.yoshuawuyts.com/linear-types-one-pager/)
- [https://without.boats/blog/changing-the-rules-of-rust/](https://without.boats/blog/changing-the-rules-of-rust/)
- [https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/)
- https://sabrinajewson.org/blog/async-drop

But the literature from MPC is also relevant.
