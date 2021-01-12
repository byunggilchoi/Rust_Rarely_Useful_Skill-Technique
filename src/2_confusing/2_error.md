# error

계속 에러가 뜨면 혈압이 오른다. 특히 에러 때문에 에러가 뜨 심각하게 혈압이 오른다.

## 대원칙

- std::result::Result<T, E>는 enum이고
  - T에는 모든 타입이 올 수 있고
  - E에는 std:error::Error trait을 구현한 타입은 모두 올 수 있음
-
