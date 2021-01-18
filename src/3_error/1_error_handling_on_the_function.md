# 에러가 발생한 함수에서 에러를 처리할 경우

튜토리얼을 보면 대부분 이렇게 처리하지만 사실 제대로 된 프로그램을 만들면 이렇게 처리할 일이 많지는 않습니다. 특히 러스트는 일반적으로 모듈 기능을 광범위하게 사용하기 때문에 특히 그렇습니다. [에러가 났을 때 기본값을 넘기는 기능](###_에러가_나면_기본값을_제공해서_넘어간다.)은

## 에러가 나면 그냥 프로그램을 종료시킨다.

이용할 경우

- 프로토타이핑이니까 에러 신경 쓸 시간에 개발을 더 하자.
  - 보통은 이렇게 시작했다가 꼼꼼하신 러스트 컴파일러께서 컴파일도 잘 안 시켜주셔서 결국은 디버깅하고 에러 메시지를 다 만들게 됩니다.
- 나는 이 프로그램이 절대 에러가 안 날 것이라는 확신이 있다.
  - 머리 속에서 암산으로 다 끝낼 수 있는 수준의 처리라면 에러가 안 나게 만드는 것도 가능하기는 한데 그걸 왜 코딩을 하시죠?

이용할 도구들

- unwrap(Result): 러스트 처음 배울 때 자주 등장해서 친숙한데 사실은 별로 좋은 에러처리 방법이 아닙니다. Result를 받아서 unwrap한 다음 Ok이면 값을 반환하고 에러이면 에러를 반환합니다.
  - unwrap_err(Result) -> E: unwrap과 반대로 작동합니다. unwrap해서 Ok이면 에러를 반환하고 에러이면 에러의 값을 Ok로 반환합니다.

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b").unwrap();
  println!("{}", content)
}
```

아래의 코드도 거의 똑같이 작동합니다.

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b");
  let content = match content {
    Ok(f) => f,
    Err(e) => panic!("{:?}", e),
  };
  println!("{}", content)
}
```

파일을 읽어서 스트링으로 변환하는데 해당 파일이 없을 수도 있겠죠. 컴파일은 우리가 던져주는 코드 외에는 아는게 없으니까요. 그래서 모든 것이 불안한 러스트 컴파일러에게는 명령 외에 명령이 실패했을 때 어떻게 할 지도 던져줘야 합니다. 그게 이 문서에 다룰 '에러 핸들링'입니다.

그 때 사용할 수 있는 가장 원시적인 수준의 방식이 `unwrap`입니다. `unwrap`은 말 그대로 앞의 결과물을 열어보는 것입니다. Result<T, E>를 받아서 T이면 T의 값을 반환하고 E이면 panic을 일으키는 거죠. 사실 아무 것도 안 한다고 보시면 됩니다.

넘겨받은 값을 unwrap했더니 에러이면 어떻게 될까요? panic이 발생하고 프로그램은 종료됩니다. panic에 대해서 모르는 분은 이 책보다는 [러스트북](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) 내용을 참고하세요. 이 문서의 내용과도 겹칩니다.

언어를 배울 때 나오는 예시 코드에서는 `unwrap`을 많이 사용합니다. 특히 입문 단계의 코드에서요. 하지만 그건 더 이상 설명을 피하기 위해서 임시방편으로 쓰는 것입니다. 실제 작동할 코드에서 unwrap을 쓰는 건 추천하지 않습니다.

## 에러가 나면 과거의 내가 보낸 메시지를 보여주면서 종료시킨다.

- 이용할 경우

  - 에러가 안 날 수는 없다는 것을 깨달았는데 에러 핸들링에 수고를 들이고 싶지는 않지만
  - 코딩하는 현재의 내가 에러 메시지를 보고 혼란(panic!)에 빠져 있을 미래의 나에게 메시지를 남기고 싶을 때

- 이용할 도구들
  - expect(Result, msg: &str) -> T: unwrap보다는 조금 더 할 수 있는게 많은 도구. Result를 받아서 unwrap을 한 다음 에러가 나오면 인자로 받은 `msg`와 에러 메시지를 함께 보여줍니다.
    - expect_err(Result, msg: &str) -> E: unwrap_err처럼 반대로 행동합니다.

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b").expect("a.b같은 파일은 없어");
  println!("{}", content)
}
```

아래의 코드도 동일하게 작동합니다.

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b");
  let content = match content {
    Ok(f) => f,
    Err(e) => panic!("a.b같은 파일은 없어: {:?}", e)
  };
  println!("{}", content)
}
```

고작 문구 하나 더 띄우는게 unwrap과 뭐가 다르냐고 생각하실 수 있습니다. 하지만 은근히 도움이 되긴 합니다.(그래도 연습용 프로젝트 이외에는 쓰지 마세요.) 러스트 컴파일러는 물론 상당히 친절해서 에러 메시지를 잘 읽어보면 원인을 상당히 많이 추론할 수 있습니다. 그렇지만 일단 에러 메시지가 뜨는 순간 우리의 이성은 에러 메시지를 "잘 읽기" 힘든 상태가 되는 경향이 있습니다.

코딩하던 과거의 내가 적어놓은 걱정이 에러와 같이 나온다면 그나마 혼란에 빠진 지금의 나에게 도움이 될 수 있습니다.

## 에러가 나면 기본값을 제공해서 넘어간다.

- 이용할 경우

  - 에러가 나더라도 프로그램을 종료시키고 싶지 않고 기본값을 제공해줄 수 있는 경우
  - 옵션값을 입력받을 수 있지만 안 받으면 그냥 기본값을 사용하는 경우
    - 서버 띄울 때 포트값이라던가 기타 등등..

- 이용할 방법들
  - unwrap_or(Result, default: T) -> T: Result를 받아서 unwrap하되 에러이면 인자로 받은 기본값을 반환
    - unwrap_or_default(Result)->T: **unwrap_or를 안 쓰고 unwrap_or_default를 쓸 일은 거의 없으니 몰라도 됨.** 그래도 책의 취지에 맞게 설명하면 Result를 받아서 Result<T, E>의 T가 가진 타입의 기본값을 반환. 예를 들어 bool의 기본값은 false, i32의 기본값은 0.
  - unwrap_or_else<F>(Result, operation: F) -> T: Result를 받아서 unwrap하되 에러이면 인자로 받은 클로저를 실행해서 결과값을 반환

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b").unwrap_or("a.b는 없지만 이걸 내용으로 쓰게나".to_string());
  println!("{}", content)
}
```

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./a.b");
  let content = match content {
    Ok(f) => f,
    Err(e) => "a.b는 없지만 이걸 내용으로 쓰게나".to_string(),
  };
  println!("{}", content)
}
```
