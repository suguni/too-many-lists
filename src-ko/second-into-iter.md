# IntoIter

컬렉션은 Rust에서 *Iterator* 트레이트를 사용하여 반복됩니다. 이는 `Drop`보다 조금 더 복잡합니다:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

여기서 새로운 것은 `type Item`입니다. 이는 모든 Iterator 구현에 Item이라는 *연관 타입*이 있음을 선언합니다. 이 경우, `next`를 호출할 때 반환할 수 있는 타입입니다.

Iterator가 `Option<Self::Item>`을 반환하는 이유는 `has_next`와 `get_next` 개념을 통합하기 위해서입니다. 다음 값이 있을 때는 `Some(value)`를 반환하고, 없을 때는 `None`을 반환합니다. 이는 API를 더 사용하기 쉽고 안전하게 만들며, `has_next`와 `get_next` 사이의 중복 검사를 피할 수 있게 합니다.

안타깝게도 Rust에는 (아직은) `yield` 문이 없어서 논리를 직접 구현해야 합니다. 또한, 각 컬렉션은 다음의 3가지 다른 종류의 반복자를 구현해야 합니다:

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

사실, 우리는 이미 List의 인터페이스를 사용하여 IntoIter를 구현할 모든 도구를 가지고 있습니다. 그냥 `pop`을 계속 호출하면 됩니다. 따라서, 우리는 IntoIter를 List의 새로운 타입 래퍼로 구현할 것입니다:

```rust ,ignore
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```

테스트를 작성해 봅시다:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

좋습니다!
