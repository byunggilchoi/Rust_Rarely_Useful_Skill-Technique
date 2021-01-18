# 많이 쓰는 방식: error_chain

결과적으로 러스트의 에러 처리는 꽤 귀찮고 반복적인 코드를 계속 만들어야 하는 문제가 있습니다. 프로그래머들이 이런 걸 그냥 둘 리가 없죠. Rustecean(러스트 사용자)들도 당연히 여러 가지 라이브러리를 통해 해결하고 있습니다. 이번 장에서는 그 중에 제가 보기에 가장 편리해 보이는 [`error_chain`](https://crates.io/crates/error-chain)을 다뤄보도록 하겠습니다.[러스트 쿡북](https://rust-lang-nursery.github.io/rust-cookbook/)에서도 사용하는 걸 봐서는 저만 편리하게 여기는 것 같지는 않네요.(둘 다 `rust-lang-nursery`에서 관리하는 리포지터리이긴 합니다. 즉 러스트 팀에서 관리하는(했던) 리포지터리입니다.)

> `rust-lang-nursery`는 [19년 말에 운영 종료를 선언](https://internals.rust-lang.org/t/rust-lang-nursery-deprecation/11205/3)했습니다. 원래 러스트 레포지터리에 올라가기에 아직 부족하지만 꽤 유용한 크레이트들을 관리하려는 목적으로 2017년 경에 만들어졌는데 crates.io가 활성화되니까 굳이 이걸 따로 관리해야하냐, 일만 많아진다는 의견들이 나왔습니다. 그래서 mdbook 등 주요 레포지터리들은 러스트 레포지터리로 옮겼습니다. 그런데... 쿡북과 error_chain은 옮기지도 않고 문을 닫지도 않았습니다. 2017년 이후 관리자가 없어서 관리가 안 된다는 이야기가 있었으나 2020년부터 다시 업데이트가 시작되었거든요.(그런데 리포지터리를 안 옮기고 있습니다) 그래서 저도 쓰고 있습니다.
> 솔직히 언제 관리가 끊길 지 잘 모르겠는데 이 정도로 많이 쓰이는 크레이트면 러스트 문법에 넣는게 맞지 않겠나 싶긴 합니다. 러스트는 아직 유저가 적어서 많이 쓰이지만 관리가 안 되는 크레이트들이 심지어 러스트 팀이 운영하는 크레이트에도 종종 있습니다. 관리자에 지원해보세요.

## error_chain 라이브러리의 특징

- 간단한 오류처리부터 복잡한 오류처리까지 일관적인 방법을 제공합니다.
  - 이게 핵심인데 이건 기본 문법이 해줘야 하는 것 아닌가요...
- 오류의 원인(Error trait의 cause 메서드)을 체계적으로 관리하게 해줍니다.
  - 이것도 기본 문법이 해줘야 하는 것 아닌가요...

다시 말씀드리지만 `error_chain`은 뭔가 어마어마한 것을 제공하지 않습니다. '기본문법에서 왜 이걸 제공하지 않지?'라는 생각이 드는 것들을 제공합니다.

### error_chain! 매크로

`error_chain!` 매크로는 `error_chain` 라이브러리의 핵심이라고 할 수 있습니다. 아래 두 가지 기능을 한 방에 제공하기 때문입니다.

- 커스텀 에러 등 반복적으로 생성해야 하는 타입들을 생성
- From trait을 해당 커스텀 에러에 구현 -> `?`연산자 사용 가능

### Error 구조체

커스텀 에러도 Error라고 이름짓는 것은 러스트의 전통이죠. `error_chain`의 Error 구조체는 아래 두 가지로 구성되어 있는 튜플 구조체입니다.

- State: 러스트의 기본 Error에서 생성된 내용을 받아주는 역할
  - 백트레이스: 에러가 처음 발생했을 때 기록
  - Error::cause()의 결과물들이 연결된 에러체인: 처음 발생한 에러가 에러를 '버블링'해서 윗단계로 넘겨주면 단계가 올라갈 때마다 내용을 '체인'에 저장
- ErrorKind: enum. 에러의 종류를 표시

```rust
pub enum ErrorKind {
  Inner(ErrorKind), // 다른 에러체인으로 연결된다는 의미. 매크로에서 links를 쓰면 Inner(ErrorKind).
  Io(Error),        // std::io::Error 타입이라는 의미. 매크로에서 foreign_links를 쓰면 Io(Error).
  Msg(String),      // 간단한 에러로 스트링을 받았을 때의 ErrorKind.
  Custom,           // 그 외의 커스텀 에러를 받았을 때의 ErrorKind.
}
```

## 한 번 써봅시다.

주절주절 말로 써도 사실 코드를 보는게 이해가 훨씬 빠릅니다.

### 에러타입 선언하기

에러타입을 선언하기 위해서는 error_chain! 매크로를 사용합니다. 해당 메크로를 사용해서 하나의 파일이나 모듈에 크레이트 전체를 위한 에러 타입을 지정할 수 있습니다. `error_chain`의 [공식문서](https://docs.rs/error-chain/)에 나오는 코드로 살펴보겠습니다.

```rust
error_chain! {
    // 아래 types는 기본값입니다. 코딩하지 않아도 자동으로 설정됩니다.
    types {
        Error, ErrorKind, ResultExt, Result;  // 이름을 이렇게 짓는 것은 전통이라고 생각합니다.
    }
    // 에러체인들이 여러 개가 있을 수도 있겠죠. links는 그 에러체인들 사이를 연결하는 방법입니다.
    // 역시 아래 내용은 적지 않아도 됩니다.
    links {
        Another(other_error::Error, other_error::ErrorKind) #[cfg(unix)];
    }
    // error_chain!으로 정의하지 않은 에러들도 있을 수 있습니다. foreign_links는 그러한 에러를 연결하는 방법입니다.
    // 역시 아래 내용은 적지 않아도 됩니다.
    foreign_links {
        Fmt(::std::fmt::Error);
        Io(::std::io::Error) #[cfg(unix)];
    }
    // ErrorKind에 들어갈 내용들을 더 정의할 수도 있습니다.
    // 여기에 정의해두면 ErrorKind에 들어가서 해당 에러 종류를 선택할 수 있습니다.
    // 각각의 에러에서 정해진 description, display 메서드를 사용해서 자체 메시지를 띄울 수 있습니다.
    // 역시 아래 내용은 적지 않아도 됩니다.
    errors {
        InvalidToolchainName(t: String) {
            description("invalid toolchain name")
            display("invalid toolchain name: '{}'", t)
        }
        UnknownToolchainVersion(v: String) {
            description("unknown toolchain version"),
            display("unknown toolchain version: '{}'", v),
        }
    }
    // skip_msg_variant가 남아있으면 Error::Msg를 사용해서 간단한 메시지를 띄우는 에러를 처리할 수 있습니다.
    skip_msg_variant
    // 커스텀 에러타입들을 나열하면 모두 ErrorKind에 포함됩니다.
    FooError,
    BarError,
}
```

### 에러 타입 사용하기

`error_chain`에서 만들어준 커스텀 에러를 사용해봅시다. 여기서는 3가지 방법을 사용할 것입니다.

- .into(): 각 ErrorKind 타입에서 사용할 수 있는 메서드
- `?`: 위의 메서드를 단순화한 연산자
- bail!: 같은 기능을 좀 더 간결하게 쓸 수 있는 매크로

러스트의 문법은 늘 같은 기능을 구현하는 방법이 너무 많다는 느낌이 듭니다.

#### .into() 메서드

```rust
error_chain! {
    errors { FooError }
}

fn foo() -> Result<()> {
    Err(ErrorKind::FooError.into())
}

fn bar() -> Result<()> {
    Err("bar error!".into())
}
```

`.into()` 메서드는 ErrorKind(스트링 포함)를 받아서 매크로에서 정한 Error 타입으로 변환합니다. 보통은 아래와 같이 줄여서 씁니다.

#### ? 연산자

```rust
error_chain! {
    errors { FooError }
}

fn foo() -> Result<()> {
    Ok(Err(ErrorKind::FooError)?)
}

fn bar() -> Result<()> {
    Ok(Err("bar error!")?)
}
```

`?`앞이 에러가 나올 경우에는 매크로에서 정한 Error로 처리하는 겁니다. 에러가 아니면 Ok로 처리하고요.

#### bail! 매크로

```rust
error_chain! {
    errors { FooError }
}

fn foo() -> Result<()> {
    if true {
      bail!{ErrorKind::FooError};
    } else {
      Ok(())
    }
}

fn bar() -> Result<()> {
    if true {
      bail!{"bar error!"};
    } else {
      Ok(())
    }
}
```

이 예제에서는 엄청난 효과가 있다고 느끼기 힘들지만 match 구문 등에서 사용할 때 꽤 유용합니다. 그리고 직관적이죠. 앞에서는 왜 Ok()로 `?` 연산자를 둘러싸는지 잘 이해가 안 가니까요. 그럼에도 너무 종류가 많다는 생각은 듭니다.(엄격하게 보면 `error_chain`이 외부 라이브러리긴 합니다만.)

### 에러들을 체이닝해봅시다.

라이브러리 이름이 `error_chain`이니까 에러들을 하나씩 쓰는 게 목적은 아니겠죠?

- Result 타입에서 chain_err() 메서드 사용
  - Result가 Err이면 chain_err은 즉각 인자로 들어온 클로져를 평가해서 ErrorKind로 변환가능한 결과를 반환
- Error 타입에서

## 예시1: main에서 에러 다루기

`error_chain`을 사용하기에 아주 좋은 사례입니다. 물론 std::io::Error, `?` 연산자를 쓰면 어렵지 않게 처리가 가능합니다만.

러스트 쿡북의 [에러 핸들링](https://rust-lang-nursery.github.io/rust-cookbook/errors/handle.html#handle-errors-correctly-in-main)에서 가져온 예시를 봅시다.

```rust
use error_chain::error_chain;     // error_chain이 2017년~2019년 동안 관리가 잘 안 되어서 예전 방식의 extern crate을 써놓은 예시들이 꽤 남아있는데 그냥 use쓰면 됩니다.

use std::fs::File;
use std::io::Read;

error_chain!{                     // 매크로를 선언해서 커스텀 타입(Error, Result, ErrorKind, ResultExt)을 만듭니다.
    foreign_links {               // foreign_links를 통해 두 가지 에러를 ErrorKind에 포함시킵니다.
        Io(std::io::Error);
        ParseInt(::std::num::ParseIntError);
    }
}

fn read_uptime() -> Result<u64> {
    let mut uptime = String::new();
    File::open("/proc/uptime")?       // 에러 가능 지점1: std::io::Error
      .read_to_string(&mut uptime)?;  // 에러 가능 지점2: std::io::Error

    Ok(uptime
        .split('.')
        .next()                             // 옵션 반환
        .ok_or("Cannot parse uptime data")? // 반환된 옵션을 Result로 변환하면서 None이면 에러 반환->에러 가능 지점3: Err(E)-> 별도의 msg가 없음->스트링을 받아서 Msg로 가지고 버블링
        .parse()?)                          // 에러 가능 지점4: std::num::ParseIntError
}

fn main() {
    match read_uptime() {
        Ok(uptime) => println!("uptime: {} seconds", uptime), // 아무 에러가 없으면 parse()의 결과에 따라 u64타입의 숫자가 uptime에 들어감
        Err(err) => eprintln!("error: {}", err),              // 위의 네 가지 에러 중 하나가 걸리면 state에 내용을 가지고 올라와서 err에 들어감
    };
}
```

## 예시2: 에러의 타입이 바뀌어도 에러가 없어지진 않습니다

`error_chain`을 쓰면 함수에서 에러의 타입을 간단하게 변환할 수 있습니다. 물론 그 전에 ErrorKind에 해당 에러타입을 추가해둬야겠죠.
아래 예시도 러스트 쿡북의 예시를 가져왔습니다. [앞 장](./3_error/3_using_custom_error.md)에서 다뤘던 커스텀에러와 똑같은 방법인데 커스텀 에러, 커스텀 Result를 따로 지정하지 않아도 되니 훨씬 간단해집니다.

```rust
// reqwest를 사용하기 때문에 해당 크레이트를 Cargo.toml에 넣어야합니다.
use error_chain::error_chain;

error_chain! {
    foreign_links {
        Io(std::io::Error);
        Reqwest(reqwest::Error);                  // 이런 식으로 새로운 라이브러리에서 만들어놓은 Error 타입을 추가하면 됩니다.
        ParseIntError(std::num::ParseIntError);
    }
    errors { RandomResponseError(t: String) }     // 만들어 쓰실 커스텀 에러는 여기에 추가하면 됩니다.
}

fn parse_response(response: reqwest::blocking::Response) -> Result<u32> {
  let mut body = response.text()?;                // 에러가능위치1: reqwest::Error
  body.pop();
  body
    .parse::<u32>()
    .chain_err(|| ErrorKind::RandomResponseError(body)) // parse 메서드의 결과물은 Result인데 여기에 .chain_err()를 체이닝했습니다. .chain_err의 인자에 들어간 ErrorKind의 원소가 에러 타입이 됩니다. 에러가능위치2: RandomResponseError
    // map_err 메서드를 써서 아래와 같이 적어도 되겠죠.
    // body
    //  .parse::<u32>()
    //  .map_err(|e| Error::with_chain(e, ErrorKind::RandomResponseError(body)))
}

fn run() -> Result<()> {
  let url =
    format!("https://www.random.org/integers/?num=1&min=0&max=10&col=1&base=10&format=plain");
  let response = reqwest::blocking::get(&url)?;         // 에러가능위치3: reqwest::Error
  let random_value: u32 = parse_response(response)?;    // 에러1이나 2가 나왔으면 에러가 나오겠죠. 여기서의 에러타입은 두 개의 타입을 모두 가질 수 있는 Error가 됩니다.
  println!("a random number between 0 and 10: {}", random_value);
  Ok(())
}

fn main() {
  if let Err(error) = run() {                                               // run()의 결과가 에러 1, 2, 3 중 하나 때문에 Err(error)일 때
    match *error.kind() {
      ErrorKind::Io(_) => println!("Standard IO error: {:?}", error),       // 에러의 종류를 불러내는 메서드를 써서 *error.kind()가 무엇인지에 따라서 다르게 처리합니다. 앞 장에서 버블링해서 올린 에러를 어떻게 처리할 지를 다룬 내용과 동일한 방식입니다.
      ErrorKind::Reqwest(_) => println!("Reqwest error: {:?}", error),
      ErrorKind::ParseIntError(_) => println!("Standard parse int error: {:?}", error),
      ErrorKind::RandomResponseError(_) => println!("User defined error: {:?}", error),
      _ => println!("Other error: {:?}", error),
    }
  }
}

```

## 복잡한 오류에서 백트레이스 확보하기

여러 겹으로 쌓인 에러 체인은 iterator처럼 출력 가능한데 들어간 순서와 반대로 출력됩니다. 스택처럼요.

```rust
use error_chain::error_chain;
use serde::Deserialize;

use std::fmt;

error_chain! {
    foreign_links {
        Reader(csv::Error);               // 에러 타입에 포함되어야 할 에러를 적습니다
    }
}

#[derive(Debug, Deserialize)]
struct Rgb {
    red: u8,
    blue: u8,
    green: u8,
}

impl Rgb {
    fn from_reader(csv_data: &[u8]) -> Result<Rgb> {
        let color: Rgb = csv::Reader::from_reader(csv_data)
            .deserialize()
            .nth(0)
            .ok_or("Cannot deserialize the first CSV record")?  // Option을 Result로 변환. None이면 에러 메시지1 추가
            .chain_err(|| "Cannot deserialize RGB color")?;     // Result에서 에러가 나올 경우 에러 메시지2 추가

        Ok(color)
    }
}

impl fmt::UpperHex for Rgb {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let hexa = u32::from(self.red) << 16 | u32::from(self.blue) << 8 | u32::from(self.green);
        write!(f, "{:X}", hexa)
    }
}

fn run() -> Result<()> {
    let csv = "red,blue,green
102,256,204";

    let rgb = Rgb::from_reader(csv.as_bytes())                // 에러1이 발생할 경우 여기서 에러
      .chain_err(|| "Cannot read CSV data")?;                 // 그러면 여기서 에러 메시지3 추가
    println!("{:?} to hexadecimal #{:X}", rgb, rgb);

    Ok(())
}

fn main() {
    if let Err(ref errors) = run() {                          // run()에서 에러가 발생하면
        eprintln!("Error level - description");               // 에러 메시지 헤드 출력
        errors
            .iter()
            .enumerate()
            .for_each(|(index, error)| eprintln!("└> {} - {}", index, error));  // 에러 메시지 3 출력->에러 메시지 2 출력 -> 에러메시지 1 출력 -> 기본 에러메시지 출력

        if let Some(backtrace) = errors.backtrace() {
            eprintln!("{:?}", backtrace);                     // 백트레이스 나머지가 있으면 여기서 출력(RUST_BACKTRACE=1일 경우)
        }
    }
}

```
