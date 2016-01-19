# 4.9. Borrow 와 AsRef

[`Borrow`][borrow] 와 [`AsRef`][asref] 트레잇은 매우 유사하지만 다릅니다. 두 개의 트레잇이 어떤것을 의미하는지에 대해 상기시켜 보도록 합시다.

[borrow]: ../std/borrow/trait.Borrow.html
[asref]: ../std/convert/trait.AsRef.html

# Borrow

`Borrow` 트레잇은 자료구조를 작성할 때 사용 됩니다. 그리고 소유한 타입과 빌린 타입을 같은 목적으로 사용하길 원합니다.

예를 들어, [`HashMap`][hashmap] 은 `Borrow` 를 사용하는 [`get` method][get] 를 갖습니다.

```rust,ignore
fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where K: Borrow<Q>,
          Q: Hash + Eq
```

[hashmap]: ../std/collections/struct.HashMap.html
[get]: ../std/collections/struct.HashMap.html#method.get

시그너쳐가 다소 복잡하지만, 여기서 관심있는 부분은 `K` 파라미터 이다.
이것은 `HashMap` 자신의 파라미터를 가르킨다:

```rust,ignore
struct HashMap<K, V, S = RandomState> {
```

`K` 파라미터는 `HashMap`  이 사용하는 _키_ 입니다. `get()` 의 시그너처를 다시 보면,  키가 `Borrow<Q>` 를 구현했을 때 `get()` 을 사용할 수 있습니다. 이와 같은 방식으로, `String` 을 키로 사용하는 `HashMap` 을 만들지만, 검색할 대는 `&str`을 사용합니다:

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("Foo".to_string(), 42);

assert_eq!(map.get("Foo"), Some(&42));
```

표준 라이브러리는 `impl Borrow<str> for String` 를 갖고 있기 때문 입니다.

대부분의 타입에 대해서, 소유하거나 빌린 타입을 취할 때, `&T` 를 사용하면 충분합니다.
`Borrow` 가 효과적인 한 가지 경우는 한가지 이상의 빌린 값이 있는 경우 입니다.
이것은 참조나 슬라이스일 때 특히 그렇습니다: `T` 혹은 `&mut T` 를 취할 수 있습니다.
이 타입들을 받길 원한다면 `Borrow` 가 준비되어 있습니다:

```rust
use std::borrow::Borrow;
use std::fmt::Display;

fn foo<T: Borrow<i32> + Display>(a: T) {
    println!("a is borrowed: {}", a);
}

let mut i = 5;

foo(&i);
foo(&mut i);
```

위 코드는 `a is borrowed: 5` 를 두 번 출력할 것 입니다.

# AsRef

`AsRef` 트레잇은 변환 트레잇 입니다. 제너릭 코드에서 어떤 값을 참조로 변환하는데 사용됩니다. 아래와 같습니다:

```rust
let s = "Hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
    let slice = s.as_ref();
}
```

# 어떤 것을 사용할 까요?

어떻게 같은지에 대해서는 위에서 봤습니다: 어떤 타입에 대한 소유되거나 빌린 타입을 처리합니다. 그러나 둘은 약간 다릅니다.

다른 종류의 빌림에 대해 추상화 하고 싶거나, 해싱이나 비교와 같이 소유하거나 빌린 값을 동일한 방식으로 다루는 자료구조를 만들 때 `Borrow`를 선택합니다.

제너릭 코드를 작성할 때 어떤 것을 직접 참조로 바꾸고 싶을 때 `AsRef` 를 선택합니다.
