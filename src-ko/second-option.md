# Option 사용하기

특히 주의 깊은 독자들은 우리가 실제로 매우 나쁜 버전의 Option을 재발명했다는 것을 눈치챘을지도 모릅니다:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link는 그냥 `Option<Box<Node>>`입니다. 이제, `Option<Box<Node>>`를 모든 곳에 쓰지 않아도 되니 좋고, `pop`과는 달리 이를 외부에 노출하지 않기 때문에 괜찮을 수도 있습니다. 그러나 Option에는 우리가 수동으로 구현했던 몇 가지 *정말 좋은* 메서드가 있습니다. 그러지 *말고*, 모든 것을 Option으로 대체해 봅시다. 먼저, 단순히 모든 것을 Some과 None으로 변경해 보겠습니다:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

첫째로, `mem::replace(&mut option, None)`는 매우 일반적인 관용구여서 Option이 이를 메서드로 만들어 놓았습니다: `take`.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

둘째로, `match option { None => None, Some(x) => Some(y) }`는 매우 일반적인 관용구여서 이를 `map`이라고 부릅니다. `map`은 `Some(x)`의 `x`에 실행할 함수를 받아 `Some(y)`의 `y`를 생성합니다. 우리는 올바른 `fn`을 작성하고 이를 `map`에 전달할 수 있지만, 인라인으로 무엇을 할지 작성하고 싶습니다.

이를 수행하는 방법은 *클로저*를 사용하는 것입니다. 클로저 추가 슈퍼 파워를 가진 익명 함수이고, 클로저 *외부*에 있는 지역 변수 외부를 참조할 수 있습니다! 이는 다양한 조건부 로직을 수행하는 데 매우 유용합니다. 우리가 `match`를 사용하는 유일한 곳은 `pop`이므로 이를 다시 작성해 봅시다:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

아, 훨씬 낫습니다. 무언가를 깨뜨리지 않았는지 확인해 봅시다:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

좋습니다! 이제 코드의 *동작*을 실제로 개선해 봅시다.
