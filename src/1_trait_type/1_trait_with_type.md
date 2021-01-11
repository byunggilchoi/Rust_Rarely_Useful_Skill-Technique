# Trait과 Type을 함께 사용하는 방법들

## `type` 키워드

두 가지 경우에 사용됩니다. 원래는 타입 별칭(type alias)을 달기 위해서 있는 건데 '연관 타입(associaed type)' 구현에도 사용합니다.

우리가 보기에는 똑같은 문법인데 컴파일러는 다르게 처리하는 겁니다.(swift 같은 언어에는 `AssociatedType`같은 키워드가 따로 있습니다. swift 2.2 이전에는 문법의 모양을 더 중요시했는지 `typealias` 키워드였습니다만...)

이번 장에서는 두 가지 사용처를 먼저 보고 왜 그렇게 사용하는지 생각해보겠습니다.

### 타입 별칭(type alias) 작성: 기본 기능

타입 별칭이란 어떠한 타입을 정의하고 거기에 이름을 붙이는 것입니다. 즉 두 개는 서로 동치입니다.

- 길고 복잡한 타입을 쓸 때 별칭으로 정의해서 짧게 쓰는 것이 일차 목적
- 그러면서도 원래 타입에서 사용할 수 있는 기능을 그대로 사용 가능

```rust
fn main() {
// 이렇게 타입 별칭을 정의하면
type Kilometers = i32;
// 이렇게 우변 대신 별칭인 좌변을 타입으로 적을 수 있음.
// 포함관계가 아니라 Kilomters와 i32는 같은 타입.
let y: Kilometers = 5;
let x: i32 = 5;
// 같은 타입이니까 연산도 가능.
println!("x + y = {}", x + y);

// 당연히 더 복잡한 타입에도 별칭을 붙일 수 있으며 이게 이 기능이 있는 이유
type Thunk = Box<dyn Fn() + Send + 'static>;
fn takes_long_type(f: Thunk) {
      // --snip--
}
}
```

어디서 활용되고 있냐하면 바로 io::Result입니다.

Result<T> = Result<T, std::io::Error>;

그래서 io::Result 타입을 사용하면 Result의 에러 타입을 정의할 필요가 없습니다. io::Error로 지정되어 있는 Result 타입의 별칭이니까요.

### 연관 타입(associated type) 작성: 응용 기능

연관 타입이란 일종의 타입 플레이스홀더입니다. 플레이스홀더로 지정되었을 때는 무슨 타입인지 모릅니다. 나중에 (`impl`키워드를 써서) 구현할 때 무슨 타입인지 정하면 됩니다.

- `struct`나 `trait` 키워드로 정의할 때 임의의 타입을 지정해놓는(placeholder) 목적으로 사용.

```rust
pub trait Iterator {
    // 플레이스홀더를 지정해놓음
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

}
```

- `impl` 키워드로 struct나 trait의 구현할 때 type도 **타입 별칭** 방식으로 구현해야 함

```rust
struct Counter {}

pub trait Iterator {
    // 플레이스홀더를 지정해놓음
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
impl Iterator for Counter {
    // 플레이스홀더로 잡아놓은 타입->
    // 타입별칭과 같은 방법을 써서 연관 타입을 구현
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
        Some(5)
    }}
```

### 컴파일러는 어떻게 구분할까?

예시 코드를 하나 만들어서 컴파일러가 알려주는 에러들을 살펴봅시다. 1.49기준으로 나오는 에러들입니다.

```rust
// free type alias without body
// 즉 type 키워드가 type alias로 인식되었고
// type alias는 바디가 없으면 안 됩니다.
// type A;
// type alias로 인식되었고 에러가 나지 않습니다.
type B = i32;

fn c() {
  // type alias로 인식되었고 에러가 나지 않습니다.
  type D = i32;
  // type alias로 인식되었고 바디가 없다고 에러가 납니다.
  // type E;
}
```

일반적인 환경에서나 함수에서 `type` 키워드가 쓰이면 모두 type alias로 인식됩니다. type alias는 반드시 우변, body가 필요하다고 말해주고요.

구조체에서는 어떨까요?

```rust
struct F {
  // 구조체에는 identifier가 나와야 하는데
  // type이라는 키워드가 나와서 바로 에러입니다.
  // type G = i32;
  // type H;
}

// 구조체를 정의할 때는 `type`키워드 자체가 나오면 안 된다고 하네요. 구조체의 목적을 생각해보면 당연하죠.

// 마지막으로 가장 중요한 trait에서 어떻게 작동하는지 봅시다.

trait I {
  // J, K는 associated type으로 인식하고 에러가 나지 않습니다.
  // type J;
  type K;
  // L, M은 associated type으로 인식하고
  // associated type defaults are unstable 라는 에러가 나옵니다.
  // type L = i32;
  // type M = i32;
}

impl I for F {
  // associated type으로 인식하고
  // associated type은 impl에서 바디가 꼭 있어야 한다는
  // 에러가 나옵니다.
  // type J;
  // associated type으로 인식하고 에러가 나오지 않습니다
  type K = i32;
  // associated type으로 인식하고
  // associated type은 impl에서 바디가 꼭 있어야 한다는
  // 에러가 나옵니다.
  // type L;
  // associated type으로 인식하고 에러가 나오지 않지만
  // trait에서 이미 에러가 났죠.
  // type M = i32;
  // N, O는 I 트레잇 소속이 아니라서 is not a member of trait `I` 에러가 나옵니다.
  // type N;
  // type O = i32;
}
```

최대한 많은 경우의 수를 보여드렸는데 정리하면 다음과 같습니다.
|위치|인식|body|
|-|-|-|
|trait 블록|associated type|없어야 함(허용 예정)|
|trait 구현 블록|associated type|있어야 함|
|나머지|type alias|있어야 함|

trait 블록에서 `fn`키워드(메서드)의 경우에는 시그니처만 있어도 되고 함수 구현이 들어가면 기본값이 됩니다.

그런데 `type`키워드(associated type)는 아직 기본갑을 줄 수 없습니다. [1.49 기준으로 unstable하여](https://github.com/rust-lang/rust/issues/29661) 아직은 기본값을 못 주게 되어 있다고 합니다.

### 왜 이렇게 개념이 복잡한가?

일단 type alias는 많이 쓰고 associated type은 별로 쓰지 않아서 생각만큼 큰 문제는 아닙니다.

그래도 문서의 목적이 별로 유용하지는 않은 내용을 다루는 것이니까 살펴봅시다.

원래 associated type은 제네릭에서 나온 개념으로 [Rust by Example](https://doc.rust-lang.org/rust-by-example/generics/assoc_items.html)에서는 제네릭 챕터에서 설명합니다.

즉 rust에서 associated type은 제네릭 타입을 써서 표현할 수 있는 코드를 좀 더 간결하게 표현하는데 사용됩니다.
