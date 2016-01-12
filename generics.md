# 5.19. 제너릭 (Generics) - 0%

가끔, 함수나 데이터 타입을 작성할 때 다양한 인자의 타입에 대해 동작하길 원할 때가 있습니다. 운좋게도, Rust 는 제너릭 이라는 기능을 제공하는데 더 좋은 방법을 제공 힙니다. 제너릭은 타입이론에서 ‘매개변수 다형성(parametric polymorphism)’ 라고 하는데, 주어진 매개 변수(‘parametric’)에 따라 여러 형태를 갖는 (‘poly’
는 다수의, ‘morph’ 는 형태를 의미) 타입이나 함수를 의미 합니다.

타입 이론은 이만하면 됐고, 제너릭 코드를 봅시다. Rust 표준 라이브러리는 `Option<T>` 라고하는 제너릭을 제공합니다:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`<T>` 라는 것을 전에 몇 번 봤을 겁니다. 이것은 제너릭 데이터 타입을 나타냅니다.
enum 선언 안에 `T` 는 제너릭에서 사용할 타입과 바꿀 수 있습니다. 여기 `Option<T>` 를 사용하는 예가 있는데 추가적인 타입 기호가 있습니다:

```rust
let x: Option<i32> = Some(5);
```

`Option<i32>` 라고 타입 선언을 했습니다. `Option<T>` 와 얼마나 닮았는지 봅시다.
이 `Option` 의 경우, `T` 는 `i32` 에 대한 값을 갖습니다. 우측 바인딩에서 `T` 가 `5` 인 `Some(T)` 를 만들 었습니다. `5` 는 `i32` 이기 때문에 양쪽은 일치합니다. 그렇지 않으면 오류가 발생합니다:

```rust,ignore
let x: Option<f64> = Some(5);
// error: mismatched types: expected `core::option::Option<f64>`,
// found `core::option::Option<_>` (expected f64 but found integral variable)
```

오류 메세지는 `f64` 을 갖는 `Option<T>` 를 만들 수 없다는 것이 아니라, 양쪽이 일치해야 한다는 것을 의미 합니다:

```rust
let x: Option<i32> = Some(5);
let y: Option<f64> = Some(5.0f64);
```

좋습니다, 한번 정의하고 여러번 사용 합니다.

제너릭은 하나의 타입에 한정될 필요는 없습니다. rust 표준 라이브러리에서 제공하는 다른 타입을 보죠, `Result<T, E>` 입니다:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

이 타입은 두 개의 타입에 대한 제너릭 입니다: `T` and `E`
대문자는 아무 것이나 사용해도 됩니다. `Result<T, E>` 와 같이 정의 할 수도 있습니다:

```rust
enum Result<A, Z> {
    Ok(A),
    Err(Z),
}
```

관습적으로 제너릭 파라미터는 `타입`을 나타낼 때 `T` 를 사용하고, `에러`를 의미할 때 `E` 를 사용합니다. rust 에서는 신경쓰지 않아도 됩니다.

`Result<T, E>` 는 계산 결과를 돌려 주고, 제대로 동작하지 않았을 때 에러를 돌려줄 수 있도록 디자인 된 것입니다.

## 제너릭 함수(Generic functions)

비슷한 문법을 사용해서 제너릭 타입을 받는 함수를 만들 수 있습니다:

```rust
fn takes_anything<T>(x: T) {
    // do something with x
}
```

문법은 두 부분으로 나뉘는데,
`<T>` 는 "이 함수는 하나의 타입 `T`에 대한 제너릭 입니다" 라는 의미 입니다.
`x: T` 는 "x 는 `T` 타입을 갖는다" 는 의미 입니다.

여러개의 인자가 같은 제너릭 타입을 갖을 수 도 있습니다:

```rust
fn takes_two_of_the_same_things<T>(x: T, y: T) {
    // ...
}
```

여려개의 타입을 받는 버전을 작성할 수 도 있습니다:

```rust
fn takes_two_things<T, U>(x: T, y: U) {
    // ...
}
```

제너릭 함수는 ‘trait bounds’ 에서 가장 유용합니다, [traits 섹션](traits.html) 에서 다룰 에정 입니다.

## 제너릭 구조체(Generic structs)

제너릭 타입을 `struct` 에서도 사용할 수 있습니다:

```
struct Point<T> {
    x: T,
    y: T,
}

let int_origin = Point { x: 0, y: 0 };
let float_origin = Point { x: 0.0, y: 0.0 };
```

함수와 비슷하게, 제너릭 파라미터를 선언하는데 `<T>` 를 사용하고, 타입 선언에서 `x: T` 를 사용합니다.
