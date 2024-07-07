# Peek

우리가 지난번에 구현하지 않은 것 중 하나는 피크(peek)입니다. 이를 구현해 봅시다. 리스트의 머리(head)에 있는 요소의 참조를 반환하기만 하면 됩니다. 다음과 같이 시도해봅시다:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*Sigh*. 이제 무엇을 해야 할까요, Rust?

Map은 `self`를 값으로 받기 때문에 Option을 현재 위치에서 이동시킵니다. 이전에는 이를 `take`로 처리했지만, 이제는 원래 있던 위치에 그대로 두고 싶습니다. 이를 올바르게 처리하는 방법은 Option의 `as_ref` 메서드를 사용하는 것입니다. 이는 다음과 같은 정의를 가집니다:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

이는 `Option<T>`을 내부 요소에 대한 참조로 격하(demote)합니다. 명시적인 매치로 이를 직접 할 수도 있지만, *싫습니다*. 추가적인 간접 참조를 제거하기 위해 추가적인 역참조(dereference)를 해야 하지만, 다행히 `.` 연산자가 이를 처리해 줍니다.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

성공했습니다.

`as_mut`을 사용하여 이 메서드의 *가변* 버전도 만들 수 있습니다:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

간단합니다.

테스트를 잊지 마세요:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

좋습니다. 그러나 `peek_mut` 반환 값을 변경할 수 있는지 실제로 테스트하지 않았습니다. 참조가 가변적이지만 아무도 이를 변경하지 않는다면, 정말 가변성을 테스트한 것일까요? 이 `Option<&mut T>`에 `map`을 사용하여 값을 변경해 봅시다:


```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

컴파일러는 `value`가 불변이라고 불평하고 있지만, 분명히 `&mut value`를 작성했습니다. 무엇이 문제죠? 문제는 클로저의 인자를 그런 방식으로 작성해도 `value`가 가변 참조임을 지정하지 않는다는 점입니다. 대신, 이는 `|&mut value|`는 "인자는 가변 참조지만, 참조하는 값을 `value`에 복사하십시오"라는 의미의 클로저 인자와 일치할 패턴을 생성합니다. 단순히 `|value|`를 사용하면, `value`의 타입은 `&mut i32`가 되어 실제로 값을 변경할 수 있습니다.

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

훨씬 낫습니다!

