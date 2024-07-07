# IterMut

솔직히 IterMut는 정말로 대단합니다. 겉으로는 Iter와 동일해 보이지만, 공유 참조와 가변 참조의 특성 때문에 Iter는 "사소한" 반면, IterMut는 진정한 마법과도 같습니다.

Iter의 Iterator 구현에서 주요 통찰력을 얻을 수 있습니다:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

이것을 디슈가링하면 다음과 같습니다:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```


`next` 함수의 시그니처는 입력과 출력의 라이프타임 간에 아무런 제약도 설정하지 *않습니다*. 이는 `next`를 조건 없이 반복해서 호출할 수 있음을 의미합니다.


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

좋아요!

공유 참조에서는 동시에 여러 개를 가질 수 있기 때문에 *문제가 없습니다*. 그러나 가변 참조는 공존할 수 *없습니다*. 가변 참조는 독점적이어야 하기 때문입니다.

결과적으로, 안전한 코드를 사용하여 IterMut를 작성하는 것은 훨씬 더 어렵습니다. 그러나 놀랍게도 IterMut는 많은 구조에 대해 완전히 안전하게 구현될 수 있습니다.

우선 Iter 코드를 가져와 모든 것을 가변으로 변경해봅시다.

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

문제가 두 가지 다른 오류로 나뉩니다. 첫 번째 오류는 매우 명확하며, 해결 방법까지 알려줍니다! 공유 참조를 가변 참조로 업그레이드할 수 없으므로 `iter_mut`는 `&mut self`를 받아야 합니다. 단순한 복사-붙여넣기 오류입니다.

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

다른 하나는 어떨까요?

이런! 실수로 이전 섹션에서 `iter` 구현 시 오류를 만들었고, 운 좋게도 작동한 것 같습니다!

우리는 이제 Copy의 마법을 처음 경험했습니다. [소유권][ownership]을 도입할 때, 무언가를 이동하면 더 이상 사용할 수 없다고 했습니다. 일부 타입에 대해서는 이것이 완벽하게 타당합니다. 좋은 친구인 Box는 힙에 할당을 관리하며, 두 개의 코드가 메모리를 해제해야 한다고 생각하게 하고 싶지 않습니다.

그러나 다른 타입에 대해서는 이것이 *불필요*합니다. 정수는 소유권 의미가 없고, 단순한 숫자에 불과합니다. 따라서 정수는 Copy로 표시됩니다. Copy 타입은 비트 단위 복사로 완벽하게 복사할 수 있는 것으로 알려져 있습니다. 따라서 이동된 후에도 이전 값은 여전히 ​​사용 *가능합니다*. 결과적으로, 참조에서 Copy 타입을 교체 없이 이동할 수 있습니다!

Rust의 모든 숫자 기본 타입(i32, u64, bool, f32, char 등)은 Copy입니다. 모든 구성 요소가 Copy인 경우, 사용자 정의 타입도 Copy로 선언할 수 있습니다.

이 코드가 작동한 중요한 이유는 공유 참조도 Copy라는 것입니다! `&`가 Copy이므로, `Option<&>`도 *또한* Copy입니다. 그래서 `self.next.map`이 문제없이 작동한 것입니다. 이제 `&mut`는 Copy가 아니므로, 올바르게 Option을 `take`해야 합니다. (만일 &mut 를 복사한다면, 동일한 메모리 위치를 보는 두개의 &mut 가 생기고 이는 금지되는 것입니다).


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

정말 대단합니다! IterMut가 제대로 작동합니다!

테스트해봅시다:


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

정말 잘 작동합니다!

이건 정말 대단합니다.

이제 확실히 말할 수 있습니다:

우리는 단일 연결 리스트를 가져와서 리스트의 모든 요소에 대해 한 번만 가변 참조를 반환하는 코드를 구현했습니다. 이 코드는 정적으로 검증되었고, 완전히 안전하며, 복잡한 작업 없이 구현되었습니다.

이것이 가능한 이유는 몇 가지가 있습니다:

* `Option<&mut>`을 `take`하여 가변 참조에 대한 독점 접근 권한을 얻습니다. 다시 참조될 걱정이 없습니다.
* Rust는 가변 참조를 구조체의 하위 필드로 분할하는 것이 괜찮다는 것을 이해합니다. 왜냐하면 "역참조"할 방법이 없기 때문이며, 이들은 확실히 분리되어 있기 때문입니다.

이 기본 논리를 배열이나 트리에 적용하여 안전한 IterMut를 얻을 수 있습니다. 이터레이터를 앞 *과* 뒤 동시에 소비할 수 있는 DoubleEnded 이터레이터로 만들 수도 있습니다. 정말 놀랍습니다!

[ownership]: first-ownership.md
