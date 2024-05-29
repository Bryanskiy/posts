
This text aims to provide information on design and implementation details for functions delegation in generic contexts. Part of [Tracking Issue for function delegation](https://github.com/rust-lang/rust/issues/118212). For now
we consider only the delegation from free functions as this is the simplest case. However, even in this case, some alternatives arise.

## Intro

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
  foo<_, ..., _>(arg0, arg1, ..., argN)
}

```

In simplified body there is no generics and predicates, parameter types are replaced with `InferDelegation` placeholder. This information is inherited from callee during HIR ty lowering. Let's consider cases where a delegation item is expanded into free function.

## "fn to others" delegation

If the callee is not a trait method, then generics and predicates should simply be copied:

```rust
fn to_reuse<T: Clone>(_: T) {}

reuse to_reuse as bar;
// desugaring with inherited type info:
fn bar<T: Clone>(x: T) {
  to_reuse::<_>(x)
}
```

or for an inherent impl:

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
fn foo<T: Trait>(x: &T) {
  <_ as Trait>::foo(x)
}
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

- Ban delegation from free functions to methods with generics in a parent container, or even ban delegation from free functions to methods.
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

## Inference variables

It can be noted that when delegating to a generic function,  compiler generate an implicit inference variables in caller's body(look at `<_> or <_ as ...>` in above examples). We could generate generic parameter for each encountered inference variable. For the above example we get:

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

Next, we could also allow explicit inference variables in callee paths:

```rust
trait Trait {
  fn foo(&self) {}
}

reuse <_ as Trait>::foo;
// desugaring with inherited type info:
fn foo<T: Trait>(x: &T) {
  <_ as Trait>::foo(x)
}
```

and their "instantiation" by generic arguments:

```rust
trait Trait {
  fn foo(&self) {} // implicit `Self` here
}

reuse <u8 as Trait>::foo;
// desugaring with inherited type info:
fn foo<u8: Trait>(x: &u8) {
  <u8 as Trait>::foo(x)
}
```

The substituted types can also contain inference variables:

```rust
fn to_reuse<T>(_: T) {}

reuse to_reuse::<HashMap<_, _>> as bar;
// desugaring with inherited type info:
fn bar<T, U>(x: HashMap<T, U>) {
  to_reuse::<HashMap<_, _>>(x)
}
```

### Const parameters

Type parameters may not be specified in callee path because they are inferred by compiler. In contrast, constants must always be specified explicitly if the parameter is not defined by default:

```rust
fn foo<const N: i32>() {}

reuse foo as bar;
// desugaring with inherited type info:
fn bar() {
  foo() // ERROR: cannot infer the value of the const parameter `N`
}
```

Due to implementation limitations, the callee path is lowered without modifications. As a result, we get a compilation error. Therefore, we do not inherit const parameters, but they can be specified as generic arguments:
```rust
fn foo<const N: i32>() {}

reuse foo::<1> as bar;
// desugaring with inherited type info:
fn bar() {
  foo::<1>()
}
```


### Default parameters

Default parameters are also not inherited, because they are not permitted in functions. Therefore, there are 2 options here. Firstly, we can create non-default generic parameters according to inference variables:


```rust
trait Trait<T = i32> {
  fn foo(&self) {...}
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<This: Trait<T>, T>(x: &This) {
  <_ as Trait<_>>::foo(x)
}
```

Secondly, we can substitute default parameters into the signature:

```rust
trait Trait<T = i32> {
  fn foo(&self) {...}
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<This: Trait<i32>>(x: &This) {
  <_ as Trait<_>>::foo(x)
}
```

We chose the first one as it is more general.

Note: for type-relative paths type hint could be used to force default parameters:

```rust
struct S<T=i32>(T);

impl<T> S<T> {
    fn foo() {}
}

reuse S::foo; // ERROR: cannot infer type of the type parameter
reuse <S>::foo; // OK, default parameter will be used.
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
