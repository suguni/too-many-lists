# 모든 것을 제네릭으로 만들기

우리는 이미 Option과 Box에서 제네릭에 대해 다소 언급했습니다. 그러나 지금까지는 임의의 요소에 대해 실제로 제네릭인 새로운 타입을 선언하지 않았습니다.

사실, 그것은 매우 간단합니다. 지금 당장 모든 타입을 제네릭으로 만들어 봅시다:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

모든 것을 약간 더 포인터처럼 만들면, 코드가 갑자기 제네릭이 됩니다. 물론, *단순히* 이렇게만 하면 컴파일러가 매우 화를 낼 것입니다.


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

문제는 명확합니다: 우리는 `List`라는 것을 이야기하고 있지만, 그것은 더 이상 실제가 아닙니다. Option과 Box처럼, 이제 항상 `List<Something>`을 이야기해야 합니다.

그러나 모든 구현에서 사용하는 Something은 무엇일까요? List처럼, 우리의 구현도 *모든* T에 대해 작동하기를 원합니다. 따라서 List처럼, 우리의 `impl`도 포인터처럼 만들어 봅시다:


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

....이게 다입니다!


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

우리의 모든 코드는 이제 임의의 T 값에 대해 완전히 제네릭합니다. 와, 러스트는 *쉽습니다*. `new`에 특별히 주목할 만합니다. 변화가 전혀 없었습니다:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

리팩토링과 복사-붙여넣기 코딩의 수호자, Self의 영광을 만끽하십시오. 또한, 리스트 인스턴스를 생성할 때 `List<T>`를 쓰지 않는 것도 흥미롭습니다. 이 부분은 우리가 `List<T>`를 반환하는 함수에서 이를 기반으로 추론됩니다.

좋습니다, 이제 완전히 새로운 *동작*을 개선해 봅시다!

