# 그럭저럭 괜찮은 단일 연결 스택

이전 장에서는 최소 기능만 갖춘 단일 연결 스택을 작성했습니다. 하지만 몇 가지 설계 결정이 그리 좋지 않습니다. 이를 개선해 봅시다. 이 과정에서 우리는:

* 바퀴를 재발명하지 않고
* 리스트가 모든 요소 타입을 처리할 수 있도록 만들고
* 미리보기(peeking)를 추가하고
* 리스트를 반복(iterable) 가능하게 만들 것입니다.

그리고 이 과정을 통해 우리는

* 고급 Option 사용법
* 제네릭(Generic)
* 라이프타임(Lifetime)
* 반복자(Iterator)에 대해 배우게 될 것입니다.

새 파일을 `second.rs`로 추가해 봅시다:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
```

그리고 `first.rs`에 있는 모든 내용을 second.rs로 복사합니다.
