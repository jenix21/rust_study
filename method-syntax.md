# 5.16. 메소드 문법 (Method Syntax) - 0%

함수는 훌륭하지만, 데이터에 대해 여러번 함수를 호출하는 것은 어색합니다. 아래와 같은 코드를 봅시다:

```rust,ignore
baz(bar(foo)));
```

왼쪽에서 오른쪽으로 읽으려고 할 것이고, ‘baz bar foo’ 를 보게됩니다. 그런데 이건 함수가 안쪽으로 호출되어 들어가는 것이 아니라, 안에서 밖으로 나가는 것입니다: ‘foo bar baz’.
대신 이렇게 한다면 더 좋지 않을까요?

```rust,ignore
foo.bar().baz();
```

운좋게도 앞에 질문에서 추측한 것과 같이 그렇게 할 수 있습니다! Rust 는 `impl` 키워드를 통해서 이런 '메소드 호출 문법' 을 사용할 수 있도록 합니다.

# 메소드 호출 (Method calls)

어떻게 동작하는지 봅시다:

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

fn main() {
    let c = Circle { x: 0.0, y: 0.0, radius: 2.0 };
    println!("{}", c.area());
}
```

`12.566371` 을 출력 합니다.

원을 나타내는 구조체를 만들었습니다. 그리고 `impl` 블럭을 작성하고 그 안에 `area` 메소드를 정의 합니다.

메소드는 특별한 첫번째 파라미터를 받는데, 여기엔 세 개의 변종(variants)들이 있습니다:
`self`, `&self`, and `&mut self`.
첫번째 파라미터를 `foo.bar()` 에서 `foo` 라고 보면 됩니다.
세 개의 변종은 `foo` 의 가능한 형태들이라고 보면 됩니다: `self` 는 그냥 스택에 있는 값 입니다, `&self` 는 `self` 에 대한 참조 입니다, `&mut self` 는 변경 가능한 참조 입니다.
`area` 의 파라미터로 `&self` 를 받았기 때문에, 다른 파라미터 처럼 사용할 수 있습니다. `self` 가 `Circle` 라는 것을 알기 때문에 다른 구조체 처럼 `radius` 에 접근 할 수 있습니다.

소유권을 가져오는 것 보다 빌려오는 것이 좋고, 변경 가능한 참조 보다 변경 불가능한 참조를 사용하는 것이 좋기 때문에 기본적으로 `&self` 를 사용하는 것이 좋습니다. 여기 세 개의 변종에 대한 예가 있습니다:

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn reference(&self) {
       println!("taking self by reference!");
    }

    fn mutable_reference(&mut self) {
       println!("taking self by mutable reference!");
    }

    fn takes_ownership(self) {
       println!("taking ownership of self!");
    }
}
```

# 메소드 호출 연결하기 (Chaining method calls)

이제 `foo.bar()` 와 같이 메소드를 호출하는 방법을 알게 되었습니다. 그런데 처음 예제와 같은 `foo.bar().baz()` 는 어떨까요? 이것을 메소드 연결(method chaning) 이라고 하며, `self` 를 리턴함으로써 가능합니다.

```
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }

    fn grow(&self, increment: f64) -> Circle {
        Circle { x: self.x, y: self.y, radius: self.radius + increment }
    }
}

fn main() {
    let c = Circle { x: 0.0, y: 0.0, radius: 2.0 };
    println!("{}", c.area());

    let d = c.grow(2.0).area();
    println!("{}", d);
}
```

리턴 타입을 확인 해 보세요:

```
# struct Circle;
# impl Circle {
fn grow(&self) -> Circle {
# Circle } }
```

`Circle` 을 리턴 했습니다. 이 메소드로 임의의 크기로 조정된 새로운 circle 을 만들 수 있습니다.

# 연관 함수 (Associated functions)

`self` 를 인자로 받지 않는 연관 함수도 정의 할 수 있습니다.
Rust 코드에서 흔하게 볼 수 있는 패턴입니다:

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn new(x: f64, y: f64, radius: f64) -> Circle {
        Circle {
            x: x,
            y: y,
            radius: radius,
        }
    }
}

fn main() {
    let c = Circle::new(0.0, 0.0, 2.0);
}
```

이 '연관 함수'는 새로운 `Circle`을 만들어 줍니다. 연관 함수는 `ref.method()` 와 같은 문법이 아닌 `Struct::function()` 방식으로 호출 됩니다. 어떤 언어들은 연관 함수를 '정적 함수'라고 부릅니다.

# 빌더 패턴 (Builder Pattern)

사용자가 Circle 을 만들 수 있도록 하는데, 필요한 속성들만 설정 할 수 있도록 하고 싶습니다. 속성을 설정 하지 않으면 `x` 와 `y` 는 `0.0` 이 되고, `radius` 는 `1.0` 이 될 것 입니다. Rust 는 메소드 오버로딩, 이름 있는 인자, 가변 인자와 같은 문법은 제공하지 않습니다. 대신 빌더 패턴을 도입 했습니다. 이렇게 생겼습니다:

```
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

struct CircleBuilder {
    x: f64,
    y: f64,
    radius: f64,
}

impl CircleBuilder {
    fn new() -> CircleBuilder {
        CircleBuilder { x: 0.0, y: 0.0, radius: 1.0, }
    }

    fn x(&mut self, coordinate: f64) -> &mut CircleBuilder {
        self.x = coordinate;
        self
    }

    fn y(&mut self, coordinate: f64) -> &mut CircleBuilder {
        self.y = coordinate;
        self
    }

    fn radius(&mut self, radius: f64) -> &mut CircleBuilder {
        self.radius = radius;
        self
    }

    fn finalize(&self) -> Circle {
        Circle { x: self.x, y: self.y, radius: self.radius }
    }
}

fn main() {
    let c = CircleBuilder::new()
                .x(1.0)
                .y(2.0)
                .radius(2.0)
                .finalize();

    println!("area: {}", c.area());
    println!("x: {}", c.x);
    println!("y: {}", c.y);
}
```

여기서 또 다른 구조체인 `CircleBuilder` 을 만들었습니다. 거기에 빌더 메소드들을 정의했습니다.
`Circle` 에 `area()` 역시 정의 했습니다. 또 하나 `CircleBuilder` 에 `finalize()` 메소드를 정의 했구요. 이 메소드는 빌더로 부터 최종 `Circle` 을 만들어 냅니다. 이제까지 우리의 관심사를 강제 하게 위해 타입 시스템을 사용해 왔습니다: `Circle` 을 우리가 원하는 방색대로 만들도록 제약하기 위해 `CircleBuilder` 의 메소드들을 사용할 수 있습니다.
