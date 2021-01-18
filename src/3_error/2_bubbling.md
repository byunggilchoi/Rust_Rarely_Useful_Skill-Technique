# 에러를 해당 함수 밖으로 보내서 처리하는 방법

## 에러를 일단 상위 모듈로 올려서 조치를 취하고 싶다.

- 이용할 경우

  - 에러가 일어난 함수에서는 컨텍스트가 부족해서 정확한 에러 핸들링이 어려워서 발생한 에러를 호출한 함수로 올릴 때

- 이용할 방법들
  - ?: unwrap()과 같은데 Result를 받아서 에러일 때 panic을 발생시키고 시스템을 종료하는게 아니라 그 에러를 반환
    - 그러면 반환된 에러를 상위 함수가 받을 수 있음

```rust
fn main() {
  match file() {
    Ok(f) => println!("{}", f),
    Err(e) => eprintln!("error\n  {}", e),
  }
}

fn file() -> Result<T, E> {
  let url = "https://postman-echo.com/time/object";
  let result = reqwest::blocking::get(url);

  let response = match result {
    Ok(res) => res,
    Err(err) => return Err(err),
  };

  let body = response.json::<HashMap<String, i32>>();

  let json = match body {
    Ok(json) => json,
    Err(err) => return Err(err),
  };

  let date = json["years"].to_string();

  Ok(date)
}
```

## 한 함수에서 여러 종류의 에러가 발생하는 경우

러스트에서 "에러"는 특정한 타입을 이야기하는 것이 아닙니다. 공부할 때 꽤나 골치아프게 했었을 trait을 이용해서 정의됩니다. std::error::Error trait을 구현한 모든 타입은 "에러"인 거죠.

그래서 흔히 때에 맞는 에러를 만들어서 씁니다. 왠만한 라이브러리에서도 자기네 에러를 만들어 쓰고 프로그램을 만들때도 전용 에러들을 만들어서 쓰는 경우가 많습니다.

그래서 한 가지 함수에서 여러 종류의 에러가 발생할 수도 있죠. 그런데 타입 유니언이 안 되기 때문에 결과 Result 타입에 두 가지 에러 타입을 받게 할 수도 없습니다.

함수를 쪼개는 방법도 가능하겠지만 너무 귀찮은 일이고 보통은 다이내믹 디스패치를 이용합니다. `Result<T, Box<dyn std::error::Error>`와 같은 식으로요.

> dynamic dispatch: 시그니처에서 특정 타입을 지정하지 않고 trait을 지정. 해당 trait을 구현한 타입이면 모두 받을 수 있음

## 에러를 받은 상위 함수에서 할 수 있는 일들

- 사용할 경우
- 사용할 도구들
  - downcast
  - downcast_mut
  - downcast_ref
