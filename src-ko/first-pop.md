# Pop

`push`와 마찬가지로 `pop`도 목록을 변경하려고 합니다. `push`와 달리 실제로는 무언가를 반환하고자 합니다. 하지만 `pop`도 까다로운 코너 케이스를 처리해야 합니다. 목록이 비어 있으면 어떻게 할까요? 이 경우를 표현하기 위해 신뢰할 수 있는 `Option` 타입을 사용합니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>`는 존재할 수 있는 값을 나타내는 열거형입니다. `Some(T)` 또는 `None`이 될 수 있습니다. Link에서 했던 것처럼 자체 열거형을 만들 수도 있지만, 사용자가 반환 유형이 무엇인지 이해할 수 있기를 바라며, Option은 매우 보편적이어서 *누구나* 알고 있습니다. 사실, 모든 파일에서 암시적으로 범위로 가져올 뿐만 아니라 그 변형인 `Some`과 `None`(따라서 `Option::None`이라고 말할 필요가 없습니다)에도 포함될 정도로 매우 기본적인 요소입니다.

`Option<T>`의 뾰족한 부분은 Option이 실제로 T에 *제너릭*이라는 것을 나타냅니다. 즉, *모든* 유형에 대한 Option을 만들 수 있다는 뜻입니다!

그렇다면 이 `Link`가 있는데, 비어 있는지 아니면 더 있는지 어떻게 알 수 있을까요? `match`로 패턴 매칭을!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

이런, `pop`은 값을 반환해야 하는데 아직 그렇게 하지 않고 있습니다. `None`을 반환*할 수도** 있지만, 이 경우에는 함수 구현이 완료되지 않았음을 나타내기 위해 `unimplemented!()`를 반환하는 것이 더 좋을 것입니다. `unimplemented!()`은 매크로(`!`은 매크로를 나타냄)로, 이 매크로에 도달하면 프로그램을 패닉 상태로 만듭니다(\~ 제어된 방식으로 충돌).

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```
무조건 패닉은 [발산 함수][diverging]의 한 예입니다. 발산 함수는 호출자에게 반환되지 않으므로 모든 유형의 값이 예상되는 곳에서 사용할 수 있습니다. 여기서는 `Option<T>` 타입의 값 대신 `unimplemented!()`가 사용되고 있습니다.

또한 프로그램에서 `return`을 작성할 필요가 없다는 점에 유의하세요. 함수의 마지막 표현식(기본적으로 줄)은 암시적으로 그 반환값입니다. 이를 통해 아주 간단한 것을 좀 더 간결하게 표현할 수 있습니다. 다른 C 계열 언어처럼 언제든지 `return`으로 명시적으로 일찍 반환할 수 있습니다.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

어서, 러스트, 우리한테서 떨어져! 언제나 그렇듯 러스트는 우리에게 화를 냅니다. 다행히도 이번에는 모든 정보를 알려주었습니다! 기본적으로 패턴 일치는 그 내용을 새 브랜치로 이동하려고 시도하지만, 여기에는 자체 바이값이 없기 때문에 이 작업을 수행할 수 없습니다.

```text
help: consider borrowing here: `&self.head`
```

Rust는 이 문제를 해결하기 위해 `match`에 참조를 추가해야 한다고 말합니다. 🤷‍♀️ 해봅시다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

만세, 다시 컴파일! 이제 그 로직을 알아봅시다. Option을 만들고 싶으니 이를 위한 변수를 만들어 봅시다. Empty의 경우 None을 반환해야 합니다. More의 경우에는 `Some(i32)`를 반환하고 목록의 헤드를 변경해야 합니다. 그럼 기본적으로 그렇게 해보겠습니다.

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*head*

*desk*

우리는 `node`에 대한 공유 참조만 있는 상황에서 '노드'에서 벗어나려고 합니다.

We're trying to move out of `node` when all we have is a shared reference to it.

한 발 물러서서 우리가 무엇을 하려는 것인지 생각해 봐야 합니다. 우리는 다음을 하고 싶습니다:

* 목록이 비어 있는지 확인합니다.
* 비어 있으면 None을 반환합니다.
* 비어 있지 *않은* 경우
    * 목록의 head를 제거합니다.
    * 해당 `elem`을 제거합니다.
    * 리스트의 head를 `next`로 바꿉니다.
    * `Some(elem)`을 반환합니다.

핵심 인사이트는 우리가 무언가를 *제거*하고 싶다는 것, 즉 목록의 헤드를 *값으로* 가져오고 싶다는 것입니다. 하지만 `&self.head`를 통해 얻는 공유 참조를 통해서는 그렇게 할 수 없습니다. 우리는 또한 `self`에 대한 가변 참조자만 "유일하게" 가지고 있으므로, 물건을 옮길 수 있는 유일한 방법은 그것을 *바꾸는 것*뿐입니다. 다시 Empty 춤을 추는 것 같네요!

한번 해봅시다:


```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

O M G

아무런 *경고* 없이 컴파일되었습니다!!!!!

사실 저는 여기에 제 개인적인 보푸라기를 적용하려고 합니다. 이 `result` 값을 반환하도록 만들었지만 실제로는 그렇게 할 필요가 전혀 없었습니다! 함수가 마지막 표현식으로 평가되는 것처럼 모든 블록도 마지막 표현식으로 평가됩니다. 일반적으로 세미콜론으로 이 동작을 억제하여 블록이 빈 튜플인 `()`로 평가되도록 합니다. 이것은 실제로 `push`와 같이 반환값을 선언하지 않는 함수가 반환하는 값입니다.

따라서 대신 `pop`을 다음과 같이 쓸 수 있습니다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

좀 더 간결하고 관용적인 표현입니다. Link::Empty 브랜치는 평가할 표현식이 하나뿐이므로 중괄호가 완전히 사라졌습니다. 간단한 경우를 위한 멋진 속기입니다.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

좋아요, 여전히 작동합니다!



[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
