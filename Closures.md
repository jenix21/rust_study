# 5.23. 클로저 (Closures) - 10%

러스트는 익명함수 또한 사용할 수 있습니다. 익명함수는 (한 영역 밖으로는 닫혀있기 때문에) 클로저라 불리는 연결된 영역을 가지고 있습니다. 러스트는 이것을 정말 잘 지원하고 있습니다.
실제로 살펴보도록 하죠.

## Syntax

다음과 같은 클로저가 있습니다:

```rust
let plus_one = |x: i32| x + 1;

assert_eq!(2, plus_one(1));
```

우리는 `plus_one`에 바인딩을 만들어, 클로저에 그것을 할당했습니다. 클로저의 인수(arguments)는 파이프(`|`) 사이에, 몸체(body)에는 표현식을 씁니다.  `x + 1` 이 해당됩니다.
`{  }`를 기억하신다면, 우리는 여러 줄로 된 클로저 또한 만들 수 있습니다.

```rust
let plus_two = |x| {
    let mut result: i32 = x;

    result += 1;
    result += 1;

    result
};

assert_eq!(4, plus_two(2));
```

클로저가 `fn`으로 정의되는 일반 함수와 약간 다르다는 것을 눈치채셨을 겁니다. 첫번째로 클로저가 받는 인수나 반환값의 타입을 명시해줄 필요가 없다는 것입니다.
물론 명시해 줄 수 있습니다:

```rust
let plus_one = |x: i32| -> i32 { x + 1 };

assert_eq!(2, plus_one(1));
```

그렇지만 그렇게 할 필요가 없습니다. 왜 그럴까요? 기본적으로, 사람이 보다 편하기 위해서입니다. 일반 함수가 문서나 타입 추론에 도움이 되게 하기 위해 타입들을 모두 명시하는 반면에, 클로저는 익명함수로써 error-at-a-distance 종류의 문제를 일으키지 않기 때문에 좀처럼 그렇게 하지 않습니다.

두번째는 구문이 비슷하지만, 조금은 다르다는 것입니다. 간단한 비교를 해보죠.

```rust
fn  plus_one_v1   (x: i32 ) -> i32 { x + 1 }
let plus_one_v2 = |x: i32 | -> i32 { x + 1 };
let plus_one_v3 = |x: i32 |          x + 1  ;
```

조금 다르지만, 그들은 비슷합니다.

## 클로저와 영역

클로저(Closure)라 불리는 이유는 그들의 영역에 닫혀있기(close) 때문입니다.
다음을 보면:

```rust
let num = 5;
let plus_num = |x: i32| x + num;

assert_eq!(10, plus_num(5));
```

`plus_num` 이라는 클로저는
This closure, `plus_num`, refers to a `let` binding in its scope: `num`. More
specifically, it borrows the binding. If we do something that would conflict
with that binding, we get an error. Like this one:

```rust,ignore
let mut num = 5;
let plus_num = |x: i32| x + num;

let y = &mut num;
```

Which errors with:

```text
error: cannot borrow `num` as mutable because it is also borrowed as immutable
    let y = &mut num;
                 ^~~
note: previous borrow of `num` occurs here due to use in closure; the immutable
  borrow prevents subsequent moves or mutable borrows of `num` until the borrow
  ends
    let plus_num = |x| x + num;
                   ^~~~~~~~~~~
note: previous borrow ends here
fn main() {
    let mut num = 5;
    let plus_num = |x| x + num;

    let y = &mut num;
}
^
```

A verbose yet helpful error message! As it says, we can’t take a mutable borrow
on `num` because the closure is already borrowing it. If we let the closure go
out of scope, we can:

```rust
let mut num = 5;
{
    let plus_num = |x: i32| x + num;

} // plus_num goes out of scope, borrow of num ends

let y = &mut num;
```

If your closure requires it, however, Rust will take ownership and move
the environment instead. This doesn’t work:

```rust,ignore
let nums = vec![1, 2, 3];

let takes_nums = || nums;

println!("{:?}", nums);
```

We get this error:

```text
note: `nums` moved into closure environment here because it has type
  `[closure(()) -> collections::vec::Vec<i32>]`, which is non-copyable
let takes_nums = || nums;
                    ^~~~~~~
```

`Vec<T>` has ownership over its contents, and therefore, when we refer to it
in our closure, we have to take ownership of `nums`. It’s the same as if we’d
passed `nums` to a function that took ownership of it.

## `move` closures

We can force our closure to take ownership of its environment with the `move`
keyword:

```rust
let num = 5;

let owns_num = move |x: i32| x + num;
```

Now, even though the keyword is `move`, the variables follow normal move semantics.
In this case, `5` implements `Copy`, and so `owns_num` takes ownership of a copy
of `num`. So what’s the difference?

```rust
let mut num = 5;

{
    let mut add_num = |x: i32| num += x;

    add_num(5);
}

assert_eq!(10, num);
```

So in this case, our closure took a mutable reference to `num`, and then when
we called `add_num`, it mutated the underlying value, as we’d expect. We also
needed to declare `add_num` as `mut` too, because we’re mutating its
environment.

If we change to a `move` closure, it’s different:

```rust
let mut num = 5;

{
    let mut add_num = move |x: i32| num += x;

    add_num(5);
}

assert_eq!(5, num);
```

We only get `5`. Rather than taking a mutable borrow out on our `num`, we took
ownership of a copy.

Another way to think about `move` closures: they give a closure its own stack
frame.  Without `move`, a closure may be tied to the stack frame that created
it, while a `move` closure is self-contained. This means that you cannot
generally return a non-`move` closure from a function, for example.

But before we talk about taking and returning closures, we should talk some more
about the way that closures are implemented. As a systems language, Rust gives
you tons of control over what your code does, and closures are no different.

## Closure implementation

Rust’s implementation of closures is a bit different than other languages. They
are effectively syntax sugar for traits. You’ll want to make sure to have read
the [traits chapter][traits] before this one, as well as the chapter on [trait
objects][trait-objects].

[traits]: traits.html
[trait-objects]: trait-objects.html

Got all that? Good.

The key to understanding how closures work under the hood is something a bit
strange: Using `()` to call a function, like `foo()`, is an overloadable
operator. From this, everything else clicks into place. In Rust, we use the
trait system to overload operators. Calling functions is no different. We have
three separate traits to overload with:

```rust
# mod foo {
pub trait Fn<Args> : FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
# }
```

You’ll notice a few differences between these traits, but a big one is `self`:
`Fn` takes `&self`, `FnMut` takes `&mut self`, and `FnOnce` takes `self`. This
covers all three kinds of `self` via the usual method call syntax. But we’ve
split them up into three traits, rather than having a single one. This gives us
a large amount of control over what kind of closures we can take.

The `|| {}` syntax for closures is sugar for these three traits. Rust will
generate a struct for the environment, `impl` the appropriate trait, and then
use it.

## Taking closures as arguments

Now that we know that closures are traits, we already know how to accept and
return closures: just like any other trait!

This also means that we can choose static vs dynamic dispatch as well. First,
let’s write a function which takes something callable, calls it, and returns
the result:

```rust
fn call_with_one<F>(some_closure: F) -> i32
    where F : Fn(i32) -> i32 {

    some_closure(1)
}

let answer = call_with_one(|x| x + 2);

assert_eq!(3, answer);
```

We pass our closure, `|x| x + 2`, to `call_with_one`. It just does what it
suggests: it calls the closure, giving it `1` as an argument.

Let’s examine the signature of `call_with_one` in more depth:

```rust
fn call_with_one<F>(some_closure: F) -> i32
#    where F : Fn(i32) -> i32 {
#    some_closure(1) }
```

We take one parameter, and it has the type `F`. We also return a `i32`. This part
isn’t interesting. The next part is:

```rust
# fn call_with_one<F>(some_closure: F) -> i32
    where F : Fn(i32) -> i32 {
#   some_closure(1) }
```

Because `Fn` is a trait, we can bound our generic with it. In this case, our closure
takes a `i32` as an argument and returns an `i32`, and so the generic bound we use
is `Fn(i32) -> i32`.

There’s one other key point here: because we’re bounding a generic with a
trait, this will get monomorphized, and therefore, we’ll be doing static
dispatch into the closure. That’s pretty neat. In many languages, closures are
inherently heap allocated, and will always involve dynamic dispatch. In Rust,
we can stack allocate our closure environment, and statically dispatch the
call. This happens quite often with iterators and their adapters, which often
take closures as arguments.

Of course, if we want dynamic dispatch, we can get that too. A trait object
handles this case, as usual:

```rust
fn call_with_one(some_closure: &Fn(i32) -> i32) -> i32 {
    some_closure(1)
}

let answer = call_with_one(&|x| x + 2);

assert_eq!(3, answer);
```

Now we take a trait object, a `&Fn`. And we have to make a reference
to our closure when we pass it to `call_with_one`, so we use `&||`.

## Function pointers and closures

A function pointer is kind of like a closure that has no environment. As such,
you can pass a function pointer to any function expecting a closure argument,
and it will work:

```rust
fn call_with_one(some_closure: &Fn(i32) -> i32) -> i32 {
    some_closure(1)
}

fn add_one(i: i32) -> i32 {
    i + 1
}

let f = add_one;

let answer = call_with_one(&f);

assert_eq!(2, answer);
```

In this example, we don’t strictly need the intermediate variable `f`,
the name of the function works just fine too:

```ignore
let answer = call_with_one(&add_one);
```

## Returning closures

It’s very common for functional-style code to return closures in various
situations. If you try to return a closure, you may run into an error. At
first, it may seem strange, but we’ll figure it out. Here’s how you’d probably
try to return a closure from a function:

```rust,ignore
fn factory() -> (Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let answer = f(1);
assert_eq!(6, answer);
```

This gives us these long, related errors:

```text
error: the trait `core::marker::Sized` is not implemented for the type
`core::ops::Fn(i32) -> i32` [E0277]
fn factory() -> (Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~
note: `core::ops::Fn(i32) -> i32` does not have a constant size known at compile-time
fn factory() -> (Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~
error: the trait `core::marker::Sized` is not implemented for the type `core::ops::Fn(i32) -> i32` [E0277]
let f = factory();
    ^
note: `core::ops::Fn(i32) -> i32` does not have a constant size known at compile-time

let f = factory();
    ^
```

In order to return something from a function, Rust needs to know what
size the return type is. But since `Fn` is a trait, it could be various
things of various sizes: many different types can implement `Fn`. An easy
way to give something a size is to take a reference to it, as references
have a known size. So we’d write this:

```rust,ignore
fn factory() -> &(Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let answer = f(1);
assert_eq!(6, answer);
```

But we get another error:

```text
error: missing lifetime specifier [E0106]
fn factory() -> &(Fn(i32) -> i32) {
                ^~~~~~~~~~~~~~~~~
```

Right. Because we have a reference, we need to give it a lifetime. But
our `factory()` function takes no arguments, so elision doesn’t kick in
here. What lifetime can we choose? `'static`:

```rust,ignore
fn factory() -> &'static (Fn(i32) -> i32) {
    let num = 5;

    |x| x + num
}

let f = factory();

let answer = f(1);
assert_eq!(6, answer);
```

But we get another error:

```text
error: mismatched types:
 expected `&'static core::ops::Fn(i32) -> i32`,
    found `[closure <anon>:7:9: 7:20]`
(expected &-ptr,
    found closure) [E0308]
         |x| x + num
         ^~~~~~~~~~~

```

This error is letting us know that we don’t have a `&'static Fn(i32) -> i32`,
we have a `[closure <anon>:7:9: 7:20]`. Wait, what?

Because each closure generates its own environment `struct` and implementation
of `Fn` and friends, these types are anonymous. They exist just solely for
this closure. So Rust shows them as `closure <anon>`, rather than some
autogenerated name.

But why doesn’t our closure implement `&'static Fn`? Well, as we discussed before,
closures borrow their environment. And in this case, our environment is based
on a stack-allocated `5`, the `num` variable binding. So the borrow has a lifetime
of the stack frame. So if we returned this closure, the function call would be
over, the stack frame would go away, and our closure is capturing an environment
of garbage memory!

So what to do? This _almost_ works:

```rust,ignore
fn factory() -> Box<Fn(i32) -> i32> {
    let num = 5;

    Box::new(|x| x + num)
}
# fn main() {
let f = factory();

let answer = f(1);
assert_eq!(6, answer);
# }
```

We use a trait object, by `Box`ing up the `Fn`. There’s just one last problem:

```text
error: closure may outlive the current function, but it borrows `num`,
which is owned by the current function [E0373]
Box::new(|x| x + num)
         ^~~~~~~~~~~
```

We still have a reference to the parent stack frame. With one last fix, we can
make this work:

```rust
fn factory() -> Box<Fn(i32) -> i32> {
    let num = 5;

    Box::new(move |x| x + num)
}
# fn main() {
let f = factory();

let answer = f(1);
assert_eq!(6, answer);
# }
```

By making the inner closure a `move Fn`, we create a new stack frame for our
closure. By `Box`ing it up, we’ve given it a known size, and allowing it to
escape our stack frame.
