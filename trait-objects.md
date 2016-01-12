# 5.23. 트레잇 객체 (Trait Objects)

다형성과 관련된 코드가 실행될 때, 어떤 버전의 코드가 실행될지 결정하는 메카니즘이 필요합니다.
이것을 'dispatch' 라고 하며, 두 가지의 주요 형태가 있습니다: 정적 dispatch 와 동적 dispatch.
rust 는 정적 dispatch 를 선호하는데, 'trait objects' 라는 메커니즘을 통해서 동적 dispatch 역시 지원합니다.

## 배경

이 장에서 트레잇과 몇 개의 구현체가 필요합니다.
`Foo` 라는 간단한 코드를 만들어 봅시다. `String` 을 리턴하는 하나의 메소드를 갖고 있습니다.

```rust
trait Foo {
    fn method(&self) -> String;
}
```

`u8`과 `String` 에 대해 이 트레잇을 구현 할 것입니다.

```rust
# trait Foo { fn method(&self) -> String; }
impl Foo for u8 {
    fn method(&self) -> String { format!("u8: {}", *self) }
}

impl Foo for String {
    fn method(&self) -> String { format!("string: {}", *self) }
}
```

## 정적 dispatch

정적 dispatch 를 수행하기 위해, 이 트레잇과 트레잇 바운드를 사용할 수 있습니다:

```rust
# trait Foo { fn method(&self) -> String; }
# impl Foo for u8 { fn method(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn method(&self) -> String { format!("string: {}", *self) } }
fn do_something<T: Foo>(x: T) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something(x);
    do_something(y);
}
```

여기서 rust 는 정적 dispatch 를 수행하기 위해 단형화(monomorphization)를 사용합니다.
이 말은 rust 는 `u8` 과 `String` 을 위한 특별한 버전의 `do_something()`을 만들고, 이 메소드에 대한 호출부들을 이 구체화된 함수들에 대한 호출로 바꾼다는 것을 의미 합니다. 다르게 말하면, rust 는 이렇게 생성 합니다:

```rust
# trait Foo { fn method(&self) -> String; }
# impl Foo for u8 { fn method(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn method(&self) -> String { format!("string: {}", *self) } }
fn do_something_u8(x: u8) {
    x.method();
}

fn do_something_string(x: String) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something_u8(x);
    do_something_string(y);
}
```

이것은 아주 큰 장점을 지니는데: 함수 호출하는 부분을 컴파일 타임에 알 수 있기 때문에, 정적 dispatch 는 함수 호출을 인라인 할 수 있으며, 인라인은 좋은 최적화의 핵심 입니다. 정적 dispatch 는 빠르지만 단점이 있습니다: 바이너리에 각 타입 별로 같은 함수의 복사본들이 있기 때문에 코드가 커집니다. (code bloat)

뿐만 아니라, 컴파일러는 완벽하지 않기 때문에 "최적화" 코드가 더 느려질 수 도 있습니다.
예를 들면, 너무 과하게 함수가 인라인되면 명령어 캐시를 넘칠 수 있습니다. (캐시가 우리 주변의 모든 것을 지배합니다)
이것이 `#[inline]`과 `#[inline(always)]` 이 신중하게 사용되어야 하는 이유이고, 동적 dispatch 가 가끔 더 효율적인 이유 입니다.

그러나, 일반적인 경우 정적 dispatch 를 사용하는 것이 더 효율적이고, 항상 정적으로 dispatch 되는 wrapper 함수가 동적 dispatch 를 수행하도록 할 수 있습니다. 반대로는 되지 않으며, 정적 호출이 더 유연하다는 것을 의미 합니다.
표준 라이브러리는 이런 이유로 가능하면 정적으로 dispatch 하도록 하고 있습니다.

## 동적 dispatch

Rust 는 ‘트레잇 객채(trait objects)’ 라는 기능을 통해 동적 dispatch 를 제공 합니다.
트레잇 객체는 `&Foo` 혹은 `Box<Foo>` 와 같이 주어진 트레잇을 구현한 *어떤* 타입의 값을 저장하는 평범한 값들이며, 정확한 타입은 런타입에 결정 될 수 있습니다.

트레잇 객체는 트레잇을 구현한 구체적인 타입에 대한 포인터를 *캐스팅* 하거나 (예. `&x as &Foo`) *강제(coercing)* 해서 (예. `&Foo` 가 파라미터로 정의된 함수에 대해 인자로 `&x` 를 사용) 얻을 수 있습니다.

이러한 트레잇 객체에 대한 강제(coercion)나 캐스팅은 `&mut Foo` 나 `Box<Foo>` 에 대한 포인터인 `&mut T`, `Box<T>` 에 대해서도 모두 그 순간에 동작합니다. 강제와 캐스팅은 동일합니다.

이 작업은 특정 포인터의 타입에 대한 컴파일러의 지식을 ‘지우는’ 것 처럼 보입니다. 이런 이유로 트레잇 객체는 가끔 ‘타입 지우개(type erasure)’ 로 불립니다.

위 예제로 돌아가면, 트레잇 객체들을 동일한 트래잇으로 캐스팅 함으로써 동적 dispatch 를 수행할 수 있습니다.

```rust
# trait Foo { fn method(&self) -> String; }
# impl Foo for u8 { fn method(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn method(&self) -> String { format!("string: {}", *self) } }

fn do_something(x: &Foo) {
    x.method();
}

fn main() {
    let x = 5u8;
    do_something(&x as &Foo);
}
```

혹은 강제하기:

```rust
# trait Foo { fn method(&self) -> String; }
# impl Foo for u8 { fn method(&self) -> String { format!("u8: {}", *self) } }
# impl Foo for String { fn method(&self) -> String { format!("string: {}", *self) } }

fn do_something(x: &Foo) {
    x.method();
}

fn main() {
    let x = "Hello".to_string();
    do_something(&x);
}
```

트레잇 객체를 받는 함수는 `Foo` 를 구현한 각각의 타입들에 대해서 구체화 하지 않습니다: 항상 그렇지는 않지만 하나의 복사본이 생성되고 적은 코드 블로트가 발생합니다. 그러나 더 느린 가상함수 호출 비용이 발생하게 되고, 효과적으로 인라이닝(inlining) 하고 관련된 최적화가 발생하는 것을 방해합니다.

### 왜 포인터인가요?

rust 는 많은 관리되는 언어(managed language)와 다르게 기본적으로 포인터를 사용해서 객체를 할당하지 않기 때문에 타입은 다른 크기를 갖을 수 있습니다.
함수의 인자로 값을 넘기는 경우, 그리고 스택(stack)에서 값을 이동하거나 저장하기 위해 힙(heap)에 할당(그리고 해제)하는 것들과 같은 일을 하는데 있어서, 컴파일 타임에 값의 크기를 아는 것은 중요 합니다.

`Foo` 와 같은 경우, 최소한 `String` (24 바이트) 혹은 `u8` (1 바이트) 크기인 값을 필요로 할 것이며, `Foo`를 구현하는 종속적인 크레이트에서의 어떤 타입이던지 역시 최소 몇 바이트가 되어야 할 것입니다.

값을 포인터를 통해 할당한다는 것은 트레잇 객체를 넘기는 경우에 한해서는 값의 크기는 유의미하지 않다는 것이며, 단지 포인터 자체의 크기만 의미 있다는 것입니다.

### 표현(Representation)

트레잇 객체를 사용한 트리잇의 메소드들은 전통적으로 (컴파일러에 의해 생성되고 관리되는) 'vtable' 이라고 하는 함수 포인터들의 특별한 기록들을 통해서 호출할 수 있습니다.

트레잇 객체는 단순하기도 하고 복잡하기도 합니다: 그것들의 핵심 표현이나 모양(layout)은 아주 간단합니다만, 이상한 에러 메세지들도 있고 확인해봐야 할 놀라운 동작들도 있습니다.

트레잇 객체의 런타임 표현을 간단하게 살펴보도록 합시다. `std::raw` 모듈은 복잡한 빌트인 타입들과 같은 모양을 갖는 구조체들을 갖고 있습니다. [including trait objects][stdraw]:

```rust
# mod foo {
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
# }
```

[stdraw]: ../std/raw/struct.TraitObject.html

즉, `&Foo` 과 같은 트리잇 객체는 'data' 포인터와 'vtable' 포인터로 구성됩니다.

data 포인터는 트리엣 객체가 저장하고 있는 (어떤 알려지지 않은 타입 `T`의) 데이터를 가르키고 있으며, vtable 포인터는 `T` 에 대한 `Foo` 의 구현에 대응되는 vtable(가상 메소드 테이블)을 가르 킵니다.

vtable 은 본질적으로는 구현된 각 메소드들에 대한 구체적인 기계 코드 조각을 가르키는 함수 포인터들을 갖고 있는 구조체 입니다. `trait_object.method()` 와 같은 메소드 호출은 vtable 에서 정확한 포인터를 가져와서 동적으로 호출 할 것입니다. 예를 들면:

```rust,ignore
struct FooVtable {
    destructor: fn(*mut ()),
    size: usize,
    align: usize,
    method: fn(*const ()) -> String,
}

// u8:

fn call_method_on_u8(x: *const ()) -> String {
    // 컴파일러는 이 함수가 u8 을 가르키는 `x` 에 대해서만 호출되도록
    // 보장합니다.
    let byte: &u8 = unsafe { &*(x as *const u8) };

    byte.method()
}

static Foo_for_u8_vtable: FooVtable = FooVtable {
    destructor: /* compiler magic */,
    size: 1,
    align: 1,

    // cast to a function pointer
    method: call_method_on_u8 as fn(*const ()) -> String,
};


// String:

fn call_method_on_String(x: *const ()) -> String {
    // 컴파일러는 이 함수가 String 을 가르키는 `x` 에 대해서만 호출되도록
    // 보장합니다.
    let string: &String = unsafe { &*(x as *const String) };

    string.method()
}

static Foo_for_String_vtable: FooVtable = FooVtable {
    destructor: /* compiler magic */,
    // values for a 64-bit computer, halve them for 32-bit ones
    size: 24,
    align: 8,

    method: call_method_on_String as fn(*const ()) -> String,
};
```
각 vtable 의 `destructor` 필드는 vtable 의 타입에 대한 리소스들을 정리 하는데 사용되는 함수를 가르킵니다, `u8` 은 사소하지만, `String` 인 경우는 메모리를 해제 할 것입니다. 이것은 `Box<Foo>` 와 같이 트레잇 객체를 소유하는 경우, `Box` 에 대한 할당과 스코프를 벗어났을 때 내부 타입을 정리 하기 위해 필요합니다. `size` 와 `align` 필드들은 지워진 타입에 대한 크기와 정렬(alignment) 요구사항들을 저장합니다; 이것들은 기본적으로 소멸자(destructor)에 포함되기 때문에 지금은 사용되지 않지만, 트레잇 객체는 계속해서 더 유연하게 만들어 지기 때문에 나중에는 사용될 것 입니다.

`Foo` 를 구현한 몇 개의 값들이 있다고 가정할 때, `Foo` 트레잇 객체의 구조에 대한 명시적인 형태나 사용은 약간 아래와 비슷할 것입니다. (타입 불일치는 무시: 어쨌든 모두 포인터):

```rust,ignore
let a: String = "foo".to_string();
let x: u8 = 1;

// let b: &Foo = &a;
let b = TraitObject {
    // store the data
    data: &a,
    // store the methods
    vtable: &Foo_for_String_vtable
};

// let y: &Foo = x;
let y = TraitObject {
    // store the data
    data: &x,
    // store the methods
    vtable: &Foo_for_u8_vtable
};

// b.method();
(b.vtable.method)(b.data);

// y.method();
(y.vtable.method)(y.data);
```
만약 `b` 혹은 `y` 가 트레잇 객체를 소유한다면 (`Box<Foo>`), 범위를 벗어 났을 때 `(b.vtable.destructor)(b.data)` (`y`도 동일하게) 가 호출될 것입니다.
