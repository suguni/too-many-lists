# Push

이제 값을 목록에 밀어넣는 함수를 작성해 보겠습니다. `push`는 목록을 *변형*하므로 `&mut self`를 사용해야 합니다. 또한 푸시하려면 i32를 가져와야 합니다:

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

먼저 요소를 저장할 노드를 만들어야 합니다:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

`next`은 무엇인가요? 글쎄요, 이전 목록 전체! 그냥... 그렇게 하면 안 될까요?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

아니요. Rust는 우리에게 옳은 말을 하고 있지만, 그것이 정확히 무엇을 의미하는지, 어떻게 해야 하는지는 분명하지 않습니다:

> cannot move out of borrowed content

`self.head` 필드를 `next`로 옮기려고 하지만 Rust는 그렇게 하는 것을 원하지 않습니다. 이렇게 하면 차용을 종료하고 정당한 소유자에게 "반환"할 때 `self`가 부분적으로만 초기화됩니다. 앞서 말했듯이, 이는 `&mut`으로 할 수 없는 *한 가지* 일입니다: 그것은 매우 무례한 일이고, Rust는 매우 예의가 있습니다(엄청나게 위험할 수도 있지만, 그렇다고 해서 *그것* 때문에 신경을 쓰는 것은 아닙니다).

무언가를 다시 넣으면 어떨까요? 즉, 우리가 만들고 있는 노드입니다:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

주사위가 없습니다. 원칙적으로 이것은 Rust가 실제로 받아들일 수 있는 것이지만, 여러 가지 이유로 받아들이지 않을 것입니다(가장 심각한 것은 [exception safety][] 때문입니다). 우리는 head가 사라진 것을 러스트가 눈치채지 못하게 할 방법이 필요합니다. 악명 높은 러스트 해커 인디아나 존스에게 조언을 구합니다:

![Indy Prepares to mem::replace](img/indy.gif)

네, 인디는 `mem::replace` 기법을 제안합니다. 이 매우 유용한 함수를 사용하면 차용한 값을 다른 값으로 *대체*하여 값을 훔칠 수 있습니다. 파일 맨 위에 `std::mem`을 가져와서 `mem`이 로컬 범위에 오도록 해 봅시다:

```rust ,ignore
use std::mem;
```

그리고, 적절하게 사용하세요:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

여기서는 self.head를 일시적으로 Link::Empty로 `replace` 한 후 리스트의 새 헤드로 대체합니다. 거짓말하지 않겠습니다. 이것은 매우 불행한 일입니다. 안타깝게도 (현재로서는) 그렇게 해야만 합니다.

하지만 이제 `push`는 다 끝났습니다! 아마도요. 솔직히 테스트해봐야 할 것 같습니다. 지금 가장 쉬운 방법은 아마도 `pop`을 작성하고 올바른 결과가 나오는지 확인하는 것입니다.



[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
