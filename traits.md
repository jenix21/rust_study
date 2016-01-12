# 5.20. 트레잇 (Traits) - 0%

[methodsyntax][methodsyntax] 를 사용해서 함수를 호출 할 때 사용했던 `impl` 키워드에 대해 기억하시나요?

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

[methodsyntax]: method-syntax.html

트레잇도 유사한데요, 트레잇을 정의할 때 메소드 시그너처(signature) 를 사용하고, 해당 구조체에 대해 트레잇을 구현하게 됩니다. 이렇게요:

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

// 메소드 시그너처 정의
trait HasArea {
    fn area(&self) -> f64;
}

// 위에서 정의한 시그너처에 대한 구현
impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}
```

위에서 보는 것 처럼, `trait` 블럭은 `impl` 블럭과 비슷해 보이지만 시그너쳐만 정의하고 메소드 내용은 정의하지 않습니다. 트레잇을 구현할 때는 `impl Item` 이 아니라 `impl Trait for Item` 을 사용합니다.

제너릭에 제약사항을 주기 위해 트레잇을 사용할 수 있습니다. 이 함수를 보면 컴파일 되지 않고 유사한 에러를 던집니다:

```rust,ignore
fn print_area<T>(shape: T) {
    println!("This shape has an area of {}", shape.area());
}
```

rust 는 아래와 같은 메세지를 출력합니다:

```text
error: type `T` does not implement any method in scope named `area`
```

`T` 는 어떤 타입이나 될 수 있기 때문에 `area` 메소드를 구현 했는지 알 수 없습니다.
그러나 제너릭 `T` 에 'trait constraint' 를 추가 함으로써, `T` 가 `area` 메소드를 구현한다는 것을 보장할 수 있습니다:

```rust
# trait HasArea {
#     fn area(&self) -> f64;
# }
fn print_area<T: HasArea>(shape: T) {
    println!("This shape has an area of {}", shape.area());
}
```

`<T: HasArea>` 문법은 `HasArea 트레잇을 구현하는 어떤 타입`을 의미 합니다.
트레잇은 함수 타입 시그너처를 정의하기 때문에, `HasArea` 를 구현하는 어떤 타입이던 `.area()` 메소드를 갖고 있을 것이라는 것을 알 수 있습니다.

여기 동작 방식에 대한 확장된 예제가 있습니다:

```rust
trait HasArea {
    fn area(&self) -> f64;
}

struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}

struct Square {
    x: f64,
    y: f64,
    side: f64,
}

impl HasArea for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }
}

fn print_area<T: HasArea>(shape: T) {
    println!("This shape has an area of {}", shape.area());
}

fn main() {
    let c = Circle {
        x: 0.0f64,
        y: 0.0f64,
        radius: 1.0f64,
    };

    let s = Square {
        x: 0.0f64,
        y: 0.0f64,
        side: 1.0f64,
    };

    print_area(c);
    print_area(s);
}
```

이 프로그램의 출력 입니다:

```text
This shape has an area of 3.141593
This shape has an area of 1
```

`print_area` 은 제너릭 입니다. 그러나 정확한 타입을 넘겨야 합니다. 맞지 않는 타입을 넘긴다면:

```rust,ignore
print_area(5);
```

컴파일 타입에 에러를 만나게 됩니다:

```text
error: failed to find an implementation of trait main::HasArea for int
```

아직까지는 구조체에만 트레잇 구현을 추가 했지만, 어떤 타입이더라도 구현할 수 있습니다.
기술적으로는 `i32` 에 대해서 `HasArea` 를 구현 _할 수 있습니다._:

```rust
trait HasArea {
    fn area(&self) -> f64;
}

impl HasArea for i32 {
    fn area(&self) -> f64 {
        println!("this is silly");

        *self as f64
    }
}

5.area();
```

이런 기본적인 타입에 대해 메소드를 구현하는게 가능하더라도 좋은 스타일은 아닙니다.

트레잇 구현을 아무데나 추가할 수 있는 것 같아 보이지만, 이런 것을 막기 위한 두 가지 제약사항이 있습니다.
첫 번째로, 트레잇이 스코프 안에 정의되어 있지 않으면 적용할 수 없습니다.
여기 예가 있는데요: 표준 라이브러리는 `File`에 파일 입출력을 위한 추가적인 기능을 제공하는 [`Write`][write] 트레잇을 제공합니다. 기본적으로 `File`은 해당 메소드가 없습니다:

[write]: ../std/io/trait.Write.html

```rust,ignore
let mut f = std::fs::File::open("foo.txt").ok().expect("Couldn’t open foo.txt");
let result = f.write("whatever".as_bytes());
# result.unwrap(); // ignore the error
```

여기서 에러가 발생합니다:

```text
error: type `std::fs::File` does not implement any method in scope named `write`

let result = f.write(b"whatever");
               ^~~~~~~~~~~~~~~~~~
```

먼저 `use` 를 통해 `Write` 트레잇을 가져와야 합니다:

```rust,ignore
use std::io::Write;

let mut f = std::fs::File::open("foo.txt").ok().expect("Couldn’t open foo.txt");
let result = f.write("whatever".as_bytes());
# result.unwrap(); // ignore the error
```

이제 에러 없이 컴파일 될 것 입니다.

이것은 누군가 `int` 에 add 메소드를 추가 했더라도, 그 트레잇을 `use` 를 통해 가져오지 않는다면 아무런 영향이 없다는 것을 의미 합니다.

트레잇을 구현하는데 한 가지 더 제약사항이 있습니다.
`impl` 로 구현하려는 트레잇이나 타입은 작성자에 의해 정의되어야 합니다. 그래서 `i32` 타입에 대해 `HasArea` 를 구현할 수 있는데, `HasArea` 는 작성자 코드 안에 있기 때문 입니다.
그러나 러스트에서 `i32` 를 위해 제공하는 트레잇을 `Float` 에 구현 할 수 없습니다. 트레잇이나 타입이 작성자의 코드 안에는 없기 때문 입니다.

트레잇에 대해 한가지 더 있습니다: 제너릭 함수와 트레잇 바운드는 하나의 형태 (‘monomorphization’, mono: one, morph: form)를 사용하기 때문에 정적으로 dispatch 됩니다.
무엇을 의미 하는지에 대해서는 [trait objects][to] 에서 자세히 다룹니다.

[to]: trait-objects.html

# 다중 트레잇 바운드

제너릭 타입 파라미터를 트레잇으로 제한할 수 있다는 것을 봤습니다:

```rust
fn foo<T: Clone>(x: T) {
    x.clone();
}
```

여러 개의 바운드가 필요한 경우 `+` 를 사용할 수 있습니다:

```rust
use std::fmt::Debug;

fn foo<T: Clone + Debug>(x: T) {
    x.clone();
    println!("{:?}", x);
}
```

`T` 는 이제 `Clone` 과 `Debug` 가 있어야 합니다.

# Where 절

몇 개의 제너릭 타입과 약간의 트레잇 제한으로 함수를 작성하는 일은 그렇게 나쁘지 않습니다.
그러나 수가 늘어날 수록 문법은 계속 불편해 집니다:

```
use std::fmt::Debug;

fn foo<T: Clone, K: Clone + Debug>(x: T, y: K) {
    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

함수의 이름은 왼쪽에 떨어져 있고, 파라미터 리스트는 오른쪽에 떨어져 있습니다.
바운드가 점점 둘 사이를 멀어지게 합니다.

`where` 절을 사용하면 이 문제를 해결 할 수 있습니다:

```
use std::fmt::Debug;

fn foo<T: Clone, K: Clone + Debug>(x: T, y: K) {
    x.clone();
    y.clone();
    println!("{:?}", y);
}

fn bar<T, K>(x: T, y: K) where T: Clone, K: Clone + Debug {
    x.clone();
    y.clone();
    println!("{:?}", y);
}

fn main() {
    foo("Hello", "world");
    bar("Hello", "workd");
}
```

`foo()` 는 이전에 보았던 문법을 사용하고, `bar()` 는 `where` 절을 사용합니다.
타입 파라미터를 정의하고, 파라미터 리스트 다음에 바운드를 추가 합니다. 길어지면 공백을 사용 할 수도 있습니다:

```
use std::fmt::Debug;

fn bar<T, K>(x: T, y: K)
    where T: Clone,
          K: Clone + Debug {

    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

이런 유연함은 복잡한 상황에서 명료한 표현을 사용할 수 있도록 합니다.

`where` 는 단순한 문법 보다 더 강력 합니다. 예를 들면:

```
trait ConvertTo<Output> {
    fn convert(&self) -> Output;
}

impl ConvertTo<i64> for i32 {
    fn convert(&self) -> i64 { *self as i64 }
}

// can be called with T == i32
fn normal<T: ConvertTo<i64>>(x: &T) -> i64 {
    x.convert()
}

// can be called with T == i64
fn inverse<T>() -> T
        // this is using ConvertTo as if it were "ConvertFrom<i32>"
        where i32: ConvertTo<T> {
    1i32.convert()
}
```

이것은 `where`의 또 다른 기능을 보여줍니다: 왼쪽 편에 바운드 되는 타입에 타입 파라미터 `T`가 아닌, 임의의 타입을 사용할 수 있습니다. (이 경우 `i32`)

## 디폴트 메소드

트레잇의 마지막 기능이 하나 있습니다: 디폴트 메소드
예제를 보는 것이 가장 쉽습니다:

```rust
trait Foo {
    fn bar(&self);

    fn baz(&self) { println!("We called baz."); }
}
```

`Foo` 트레잇을 구현하는 측에서는 `bar()`를 구현해야 합니다만, `baz()`는 구현하지 않아도 됩니다. 정의된 기본적인 동작을 제공할 것이며, 원한다면 오버라이드 할 수 있습니다:

```rust
# trait Foo {
# fn bar(&self);
# fn baz(&self) { println!("We called baz."); }
# }
struct UseDefault;

impl Foo for UseDefault {
    fn bar(&self) { println!("We called bar."); }
}

struct OverrideDefault;

impl Foo for OverrideDefault {
    fn bar(&self) { println!("We called bar."); }

    fn baz(&self) { println!("Override baz!"); }
}

let default = UseDefault;
default.baz(); // prints "We called baz."

let over = OverrideDefault;
over.baz(); // prints "Override baz!"
```

# 상속

가끔 하나의 트레잇을 구현 하는데 또 다른 트레잇을 구현해야 하는 경우가 있습니다:

```rust
trait Foo {
    fn foo(&self);
}

trait FooBar : Foo {
    fn foobar(&self);
}
```
`FooBar` 를 구현 할 때 `Foo` 도 구현해야 합니다:

```rust
# trait Foo {
#     fn foo(&self);
# }
# trait FooBar : Foo {
#     fn foobar(&self);
# }
struct Baz;

impl Foo for Baz {
    fn foo(&self) { println!("foo"); }
}

impl FooBar for Baz {
    fn foobar(&self) { println!("foobar"); }
}
```

`Foo` 를 구현하지 않으면, 에러가 발생 합니다:

```text
error: the trait `main::Foo` is not implemented for the type `main::Baz` [E0277]
```
