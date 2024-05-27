This text aims to provide information on design and implementation details for functions delegation in generic contexts. Part of [Tracking Issue for function delegation](https://github.com/rust-lang/rust/issues/118212). For now
we consider only the delegation from free functions as this is the simplest case. However, even in this case, some alternatives arise.
## "fn to others" delegation

Let's consider cases where a delegation item is expanded into free function. If the callee is not a trait method, then generics and predicates should simply be copied:

```rust
fn to_reuse<T: Clone>(_: T) {}

reuse to_reuse as bar;
// desugaring with inherited type info:
fn bar<T: Clone>(x: T) { to_reuse::<_>(x) }
```

or

```rust
impl SomeType {
  fn to_reuse<T: Clone>(&self, x: T) {}
}

reuse SomeType::to_reuse as baz;
// desugaring with inherited type info:
fn baz<T: Clone>(fst: &SomeType, scd: T) {
  SomeType::to_reuse::<_>(fst, scd)
}
```

Trait methods contain additional `Self` parameter, which could be converted into independent type parameter:

```rust
trait Trait {
  fn foo(&self) {} // implicit `Self` here
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<T: Trait>(x: &T) { <_ as Trait>::foo(x) }
```

However, here we face a problem: what if method container also have generic parameters?

```rust
trait Trait<T> {
  fn foo<U>(&self, x: U, y: T);
}

// what should be written here and how should it be desugared??
reuse Trait<???>::foo;
```

There are three options:

- Ban delegation from free functions to a methods with generics in a parent container, or even ban delegation from free functions to a methods.
- Introduce new generic parameters in delegation item like for impls.
- Map inference variables to generic params.

We would not like to follow the first option, because any subset can always be banned as long as the feature is unstable. On the contrary, we would like to accommodate as many cases as possible within a limited syntactic budget and gather the most comprehensive user experience.

An example of the second option looks like this:

```rust
trait Trait<T> {
  fn foo<U>(&self, x: U, y: T);
}

reuse<T, U> Trait<T>::foo<U>;
```

This approach was rejected because it doesn't fit into syntax budget: additional parameters, bounds and where clauses could be very annoying.

That's why we chose the last option.

## Explicit inference variables

It can be noted that when delegating to a generic function,  compiler generate an implicit inference variables in caller's body(look at `<_> or <_ as ...>` in above examples). We could generate generic parameters according to these inference variables. For the above example we get:

```rust
trait Trait<T> {
  fn foo<U>(&self, x: U, y: T);
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<S: Trait<T>, T, U>(s: &S, x: U, y: T) {
  <_ as Trait<_>>::foo::<_>(s, x, y)
}

```

Next, we could also allow explicit inference variables and their "instantiation":

```rust
fn to_reuse<T: Clone>(_: T) {}

reuse to_reuse::<u8> as bar;
// desugaring with inherited type info:
fn bar<u8: Clone>(x: u8) { to_reuse::<u8>(x) }
```

or for a trait method:

```rust
trait Trait {
  fn foo(&self) {} // implicit `Self` here
}

reuse <u8 as Trait>::foo;
// desugaring with inherited type info:
fn foo<u8: Trait>(x: &u8) { <u8 as Trait>::foo(x) }
```

The substituted types can also contain inference variables:

```rust
fn to_reuse<T>(_: T) {}

reuse to_reuse::<HashMap<_, _>> as bar;
// desugaring with inherited type info:
fn bar<T, U>(x: HashMap<T, U>) { to_reuse::<HashMap<_, _>>(x) }
```

### disadvantages

Approach with inference variables has some disadvantages.

Firstly, the order of the parameters is determined by depth-first type traversal, which may not be obvious to users. Technically, this problem can be solved by banning instantiation of types with generic parameters, i.e., banning nested inference variables:

```rust
fn to_reuse<T>(_: T) {}
reuse to_reuse::<HashMap<_, _>> as bar; // ERROR: not allowed
```

But, again, we would not like to ban anything at this stage.

Secondly, delegating from free functions to something is an exotic case for which it is difficult to find a use.

## Implementation details

The delegation proposal aims to provide a syntactic sugar for the efficient reuse of already implemented functions. From the compiler's perspective, this means we must infer caller signature, header(any qualifiers like async, or const), generics and any other type of information from a limited syntax budget.

Ideally we would expand delegation items into regular functions before AST lowering, but analysis possibilities on AST level are limited: type information is not available, type-relative paths(`Type::name`) are not resolved. That's why we first generate a simplified function body:

```rust
fn foo<Arg0, ..., ArgN>(arg0: Arg0, ... argN: ArgN)
where
    Arg0: ...,
    ...,
{ ... }

reuse foo as bar;

// Generated hir:
fn bar(
  arg0: InferDelegation(DefId(foo), Input(0)),
  ...,
  argN: InferDelegation(DefId(foo), Input(N)),
) -> InferDelegation(DefId(foo), Output) {
  foo(arg0, arg1, ..., argN)
}

```

In simplified body there is no generics and predicates, parameter types are replaced with `InferDelegation` placeholder. This information is inherited from callee during HIR ty lowering:

1. Callee type is extracted with `FnCtxt` from the call path.
2. Callee type is traversed, corresponding parameters and predicates are collected from type definitions.
3. Inference variables are replaced with generated parameters according to collected type definitions.
4. Caller signature and predicates are instantiated with new type parameters.
