# Drop

스택을 만들고, push, pop 할 수 있으며, 모든 기능이 제대로 작동하는지 테스트까지 마쳤습니다!

목록을 정리할 때 걱정해야 하나요? 기술적으로는 전혀 그렇지 않습니다! C++와 마찬가지로 Rust는 소멸자를 사용하여 리소스가 완료되면 자동으로 정리합니다. Drop이라는 *특성*을 구현하는 타입에는 소멸자가 있습니다. 특성은 인터페이스를 뜻하는 Rust의 멋진 용어입니다. Drop 특성의 인터페이스는 다음과 같습니다:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

기본적으로 "범위를 벗어나면 잠시 시간을 주겠다"는 뜻입니다.

Drop을 구현하는 타입이 포함되어 있고 *그들의* 소멸자를 호출하기만 하면 되는 경우 실제로 Drop을 구현할 필요는 없습니다. List의 경우, head 만 drop 하면 되고, 이 head는 `Box<Node>`를 drop 하려고 *시도할* 것입니다. 이 모든 것이 자동으로 처리됩니다... 단 한 가지 문제가 있습니다.

자동 처리가 잘못될 수 있습니다.

간단한 목록을 생각해 봅시다:

```text
list -> A -> B -> C
```

`list`가 drop되면 A를 drop하려고 시도하고, 이는 다시 B를, 이는 다시 C를 drop 하려고 시도합니다. 여러분 중 일부는 당연히 긴장하고 있을 것입니다. 이것은 재귀 코드이고, 재귀 코드는 스택을 날려버릴 수 있습니다!

"이건 분명히 꼬리 재귀인데, 괜찮은 언어라면 이런 코드가 스택을 망치지 않을 것이다"라고 생각하는 분도 계실 것입니다. 사실 이것은 잘못된 생각입니다! 그 이유를 알아보기 위해 컴파일러가 하는 것처럼 목록에 대해 Drop을 수동으로 구현하여 컴파일러가 해야 하는 작업을 작성해 보겠습니다:

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```
할당 해제 *후*에는 Box 내용을 drop 할 수 *없으므로* 꼬리 재귀 방식으로 drop 할 방법이 없습니다! 대신 노드를 상자에서 끌어올리는 `List`에 대한 반복적인 드롭을 수동으로 작성해야 합니다.


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

좋아요!

----------------------

<span style="float:left">![Bonus](img/profbee.gif)</span>

## 조기 최적화를 위한 보너스 섹션!

drop의 구현은 실제로 `while let Some(_) = self.pop() { }`와 *매우* 유사하지만, 확실히 더 간단합니다. 어떻게 다르며, 정수가 아닌 다른 것을 저장하도록 목록을 일반화하기 시작하면 어떤 성능 문제가 발생할 수 있을까요?

<details>
  <summary>답변을 보려면 클릭하여 확장</summary>

Pop은 `Option<i32>`를 반환하지만, 우리 구현은 Links (`Box<Node>`)만 조작합니다. 따라서 우리의 구현은 노드에 대한 포인터만 이동하지만, pop 기반은 노드에 저장한 값을 이동합니다. 목록을 일반화하여 누군가가 이를 사용하여 VeryBigThingWithADropImpl(VBTWADI)의 인스턴스를 저장하는 경우 비용이 매우 많이 들 수 있습니다. Box는 콘텐츠의 드롭 구현을 제자리에서 실행할 수 있으므로 이 문제를 겪지 않습니다. VBTWADI는 *실제로* 배열보다 연결 리스트를 사용하는 것이 더 바람직하기 때문에 이 경우 제대로 작동하지 않는 것은 약간 실망스러운 일입니다.

두 구현의 장점을 모두 누리고 싶다면 `fn pop_node(&mut self) -> Link`라는 새로운 메서드를 추가하면 `pop`과 `drop`이 모두 깔끔하게 파생될 수 있습니다.

</details>
