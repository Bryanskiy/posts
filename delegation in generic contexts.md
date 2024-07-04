
This text aims to provide information on design and implementation details for functions delegation in generic contexts. Part of [Tracking Issue for function delegation](https://github.com/rust-lang/rust/issues/118212).

P.S. Some examples are taken from the standard library or the compiler.
Revision: 7fdefb804ec300fb605039522a7c0dfc9e7dc366
## Background: on current delegation implementation

The delegation proposal aims to provide a syntactic sugar for the efficient reuse of already implemented functions. From the compiler's perspective, this means we must infer caller signature, header(any qualifiers like async, or const), generics and any other type of information from a limited syntax budget.

Ideally we would expand delegation items into regular functions before AST->HIR lowering, but analysis possibilities on AST level are limited as type information is not available. That's why we first generate a simplified function body:

```rust
reuse callee_path::name { target_expr_template }

// desugaring:
fn name(
  arg0: InferDelegation(sig_id, Input(0)),
  ...,
  argN: InferDelegation(sig_id, Input(N)),
) -> InferDelegation(sig_id, Output) {
  callee_path(target_expr_template(arg0), arg1, ..., argN)
}
```

In simplified body:

- There is no generics and predicates.
- Parameter types are replaced with `InferDelegation` placeholder. The actual signature is inherited from callee during HIR ty lowering.
- `callee_path` must be resolved in AST level to generate the caller's header and  call arguments before lowering to HIR. The restriction is caused by the immutability of `hir::Crate`. So `callee_path` can be a path to a free function(`module::name`) or a trait method(`Trait::name` or `<Type as Trait>::name`). Type-relative paths(`Type::name`) are not supported yet.

Now, let's try extending the use of delegation to generic contexts. The following paragraphs are divided according to the delegation [parent kind](https://github.com/petrochenkov/rfcs/blob/delegation/text/0000-fn-delegation.md#parent-context) but they are listed in a different order.
## free functions delegation

Firstly, let's take a look at cases where a delegation item is expanded into free function. If the callee is a free function, then generics and predicates should simply be copied:

```rust
fn to_reuse<T: Clone>(_: T) {}

reuse to_reuse as bar;
// desugaring with inherited type info:
fn bar<T: Clone>(x: T) {
  to_reuse::<_>(x)
}
```

Trait methods contain additional `Self` parameter, which could be converted into `This: Trait`:

```rust
trait Ord: Eq + PartialOrd<Self> {
  fn min(self, other: Self) -> Self
  where
    Self: Sized {...}
}

reuse Ord::min;
// desugaring with inherited type info:
fn min<T: Ord + Sized>(v1: T, v2: T) -> T {
  <_ as Ord>::min(v1, v2)
}
```

Now let's consider the case when the parent container also has generic parameters:

```rust
trait Trait<T> {
  fn foo<U>(&self, x: U, y: T);
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<This: Trait<T>, T, U>(s: &This, x: U, y: T) {
  <_ as Trait<_>>::foo::<_>(s, x, y)
}
```

It can be noted that when lowering callee path,  compiler generate an implicit inference variables(`<_> or <_ as ...>` in above examples). Next, by carefully looking at the body of the function and its signature, a general desugaring pattern for all cases can be identified:  a generic parameter and its predicates are generated for each encountered inference variable according to the corresponding path segments resolution.
____________
Was implemented in: [#125929](https://github.com/rust-lang/rust/pull/125929)
### Generic arguments

Next, we could explicitly substitute inference variables with generic arguments:

```rust
trait Trait {
  fn foo<T>(&self, x: T) {}
}

reuse <u8 as Trait>::foo::<i8>;
// desugaring with inherited type info:
fn foo<u8: Trait>(x: &u8, y: i8) {
  <u8 as Trait>::foo::<i8>(x, y)
}
```

If the substituted type also has inference variables, then additional generic parameters have to be generated for successful inference:

```rust
fn to_reuse<T>(_: T) {}

reuse to_reuse::<HashMap<_, _>> as bar;
// desugaring with inherited type info:
fn bar<T, U>(x: HashMap<T, U>) {
  to_reuse::<HashMap<_, _>>(x)
}
```

Generic arguments support has some disadvantages. Firstly, the order of the parameters is determined by depth-first type traversal, which may not be obvious to users. Technically, this problem can be solved by banning generic types in generic arguments:

```rust
fn to_reuse<T>(_: T) {}
reuse to_reuse::<HashMap<_, _>> as bar; // ERROR: not allowed
```

But we would not like to ban anything [at this stage](#drawbacks).

Secondly, this is an exotic case for which it is difficult to find a use. One of the few similar cases I could find in the standard library was `std::vec::from_elem`.

____________
Was partially implemented in [#123958](https://github.com/rust-lang/rust/pull/123958)
## delegation in impl trait

We have several possibilities here. Firstly,  callee path can be resolved to the trait being implemented. In this case, it will be enough to take the trait method and instantiate it with impl header arguments:

```rust
// library/core/src/iter/adapters/flatten.rs

impl<I: Iterator, U: IntoIterator, F> Iterator for FlatMap<I, U, F>
where
  F: FnMut(I::Item) -> U,
{
  reuse Iterator::try_fold { self.inner }
  // desugaring with inherited type info:
  fn try_fold<Acc, Fold, R>(&mut self, init: Acc, fold: Fold) -> R
  where
    Self: Sized,
    Fold: FnMut(Acc, Self::Item) -> R,
    R: Try<Output = Acc>,
  {
    Iterator::try_fold(&mut self.inner, init, fold)
  }

  ...
}
```

Further, the callee path can be resolved to a different trait or a free function. In this case, generics and predicates will still be inherited from the method of the implemented trait:

```rust
// library/core/src/iter/adapters/zip.rs

trait ZipImpl<T, U> {
  type Item;
  fn next(&mut self) -> Option<Self::Item>;
  ..
}

impl<A, B> Iterator for Zip<A, B>
where
  A: Iterator,
  B: Iterator,
{
  reuse ZipImpl::next;
  // desugaring with inherited type info:
  fn next(&mut self) -> Option<Self::Item> {
    <_ as ZipImpl<_, _>::next(self)
  }
  ...
}
```

In this example generic parameters `T, U` in `ZipImpl::next`  do not affect the inheritance of signature and generic parameters, since they are inherited from `Iterator::next`.

There is no point in supporting generic arguments, as the implementation method must match the trait method.
____________
Was implemented in [#126161](https://github.com/rust-lang/rust/pull/126161)
## delegation in inherent impl

 The most common and natural option for inherent impl's delegation are newtypes:

```rust
// library/std/src/collections/hash/set.rs

struct HashSet<T, S = RandomState> {
  base: base::HashSet<T, S>,
}

impl<T, S> HashSet<T, S> {
  reuse base::HashSet::retain { self.base }
  ...
}
```

However, at the moment there is no support for type-relative paths, so the delegation to inherent methods is not considered in this text. Delegation from inherent methods to trait methods is a relatively rare case and is often associated with type inference problems or complex mapping of type parameters to trait parameters.

Let's start with an example:

```rust
trait Iterator {
  fn any<F>(&mut self, f: F) -> bool
  where
    Self: Sized,
    F: FnMut(Self::Item) -> bool,
  {
    ...
  }
  ...
}

// compiler/rustc_data_structures/src/unord.rs
pub struct UnordItems<T, I: Iterator<Item = T>>(I);

impl<T, I: Iterator<Item = T>> UnordItems<T, I> {
  pub fn any<F: Fn(T) -> bool>(mut self, f: F) -> bool {
    self.0.any(f)
  }
  ...
}
```

now let's try to replace `fn any`  with delegation item `reuse Iterator::any`. How will we inherit  the signature, generics and predicates of the method, in particular, how do we instantiate `Self::Item` with `I`?

Let's consider the example with own generic parameters in trait:

```rust
// library/core/src/range.rs

pub trait RangeBounds<T: ?Sized> {
  fn contains<U>(&self, item: &U) -> bool
  where
    T: PartialOrd<U>,
    U: ?Sized + PartialOrd<T>,

  {
    ...
  }
  ...
}

impl<Idx: PartialOrd<Idx>> Range<Idx> {
  pub fn contains<U>(&self, item: &U) -> bool
  where
    Idx: PartialOrd<U>,
    U: ?Sized + PartialOrd<Idx>,
  {
    <Self as RangeBounds<Idx>>::contains(self, item)
  }
}
```

How should we instantiate `T` with `Idx` here during inheritance?
### suggested solution

#### layer 1: believe in type inference

We replace each unknown parameter in the signature with a free parameter. By unknown parameters, we mean associated types of the trait, or the trait's own parameters for which no generic arguments(or type binding) have been provided:

```rust
trait Trait<T> {
  fn bar<U>(&self, _: U, _: T) {}
}

struct F;
impl<T> Trait<T> for F {}

struct S(F);

impl S {
  reuse Trait::bar { self.0 }
  // desugaring with inherited type info:
  fn bar<T, U>(&self, x: U, y: T) {
    <_ as Trait<_>>::bar::<_>(&self.0, x, y)
  }
}
```

#### layer 2: provide generic arguments(or type binding)

layer 1 will only work in the simplest cases, because an unconstrained parameter will often appear which can result in a typeck error:

```rust
impl<T, I: Iterator<Item = T>> UnordItems<T, I> {
  reuse Iterator::any { self.0 }
  // desugaring with inherited type info:
  pub fn any<A, F: Fn(A) -> bool>(mut self, f: F) -> bool {
    Iterator::any(&mut self.0, f)
    //~^ ERROR: expected a closure with arguments `(A,)`
    //          found a closure with arguments `(T,)`
  }
  ...
}
```

Therefore, we can instantiate introduced unknown parameter with generic arguments(or type binding):

```rust
impl<T, I: Iterator<Item = T>> UnordItems<T, I> {
  reuse Iterator::<Item = T>::any { self.0 }
  // desugaring with inherited type info:
  pub fn any<F: Fn(T) -> bool>(mut self, f: F) -> bool {
    Iterator::any(&mut self.0, f) // OK
  }
  ...
}
```

similarly for `RangeBounds`:

```rust
impl<Idx: PartialOrd<Idx>> Range<Idx> {
  reuse <Self as RangeBounds<Idx>>::contains;
  // desugaring with inherited type info:
  pub fn contains<U>(&self, item: &U) -> bool
  where
    Idx: PartialOrd<U>,
    U: ?Sized + PartialOrd<Idx>,
  {
    <Self as RangeBounds<Idx>>::contains(self, item)
  }
}
```

## delegation in trait

Inheritance rules for traits and inherent impls are the same.

## Lifetimes

Lifetime parameters must be declared before type and const parameters. Therefore, generic parameters sometimes need to be reordered:

```rust
trait Trait<'a, A> {
    fn foo<'b, B>(&self, x: A, y: B) {...}
}

reuse Trait::foo;
// desugaring with inherited type info:
fn foo<'a, 'b, This: Trait<'a, A>, A, B>(this: &This, x: A, y: B) {
  <_ as Trait<'_, _>>::foo::<_>(this, x, y)
}
```

## Const parameters

Type parameters and lifetimes may not be specified in callee path because they are inferred by compiler. In contrast, constants must always be specified explicitly if the parameter is not defined by default:

```rust
fn foo<const N: i32>() {}

reuse foo as bar;
// desugaring with inherited type info:
fn bar() {
  foo() // ERROR: cannot infer the value of the const parameter `N`
}
```

Due to implementation limitations, the callee path is lowered without modifications. As a result, we get a compilation error.  This will be [fixed](#on-the-implementation-of-generics-inheritance) in the future. Also, for now, if constant is specified as an generic argument, the desugaring will look like this:

```rust
fn foo<const N: i32>() {}

reuse foo::<1> as bar;
// desugaring with inherited type info:
fn bar() {
  foo::<1>()
}
```

## Default parameters

Default parameters are not inherited, because they are not permitted in functions. Therefore, there are 2 options for generics inheritance here. Firstly, we can create non-default generic parameters:

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

## On the implementation of generics inheritance

Predicates inheritance has to be done in HIR->ty lowering. The restriction is caused by the necessity of instantiation predicates with given generic arguments.

In the current implementation generics are inherited during HIR ty lowering. This leads not only to the mentioned above problems with constants, but also to more general problems with type inference:

```rust
fn foo<T>(x: i32) {}

reuse foo as bar;
// desugaring
fn bar<T>() {
  foo::<_>() // ERROR: type annotations needed
}
```

In the future, we plan to solve this problem by propagating callee parameters during AST to HIR lowering:

```rust
fn foo<T>(x: i32) {}

reuse foo as bar;
// desugaring
fn bar<T>() {
  foo<T>() // OK
}
```

## Drawbacks

Some of the cases provided in this text may look weird and perhaps should not be supported. However, we would like to accommodate as many cases as possible within a limited syntactic budget and gather the most comprehensive user experience. Any subset can always be banned as long as the feature is unstable.
