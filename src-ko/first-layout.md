# 기본 데이터 레이아웃

그렇다면 연결 리스트는 무엇일까요? 기본적으로 힙에 있는 여러 개의 데이터 조각들이 힙에 순서대로 서로를 가리키는 데이터 조각들입니다(쉿, 커널 여러분!). 연결 리스트는 절차적 프로그래머가 10피트 장대로는 건드리지 말아야 할 대상입니다, 함수형 프로그래머가 모든 것에 사용하는 것이죠. 그렇다면 함수형 프로그래머에게 연결 리스트의 정의를 물어봐야 합니다. 아마 아마 다음과 같은 정의를 내릴 것입니다:

```haskell
List a = Empty | Elem a (List a)
```

이는 대략 "List는 Empty 이거나 엘리먼트 다음에 오는 List"로 읽힙니다. 이것은 *합계 타입*으로 표현되는 재귀적 정의로, "다른 타입일 수 있는 다른 값을 가질 수 있는 타입"의 멋진 이름입니다. Rust는 합계 유형을 `enum`이라고 부릅니다! C와 유사한 언어에서 왔다면, 이것은 여러분이 알고 있고 좋아하는 열거형이지만 오버라이드 된 버전입니다. 이제 이 함수 정의를 Rust로 옮겨 보겠습니다!

지금은 단순성을 유지하기 위해 제네릭을 사용하지 않겠습니다. 부호화된 32비트 정수 저장만 지원할 예정입니다:

```rust ,ignore
// in first.rs

// pub 은 이 모듈의 바깥에서 List 를 사용하길 원한다고 얘기하는 것입니다.
pub enum List {
    Empty,
    Elem(i32, List),
}
```

*휴*, 정신이 없네요. 일단 컴파일부터 해보겠습니다:

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

글쎄요. 여러분에 대해서는 잘 모르겠지만 저는 함수형 프로그래밍 커뮤니티에 배신감을 느낍니다.

실제로 오류 메시지를 확인해보면 (배신감을 극복한 후) 실제로 rustc가 이 문제를 해결하는 방법을 정확히 알려주고 있다는 것을 알 수 있습니다:

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

좋아요, `box`? 그게 뭐죠? 구글에서 `rust box` 를 검색해 봅시다...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

여기 덜...

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation.
> See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

*링크 클릭*

> `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
>
> Examples
>
> Creating a box:
>
> `let x = Box::new(5);`
>
> Creating a recursive data structure:
>
```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```rust ,ignore
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Recursive structures must be boxed, because if the definition of Cons looked like this:
>
> `Cons(T, List<T>),`
>
> It wouldn't work. This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons. By introducing a Box, which has a defined size, we know how big Cons needs to be.

와우, 어. 제가 지금까지 본 문서 중 가장 관련성이 높고 유용한 문서입니다. 말 그대로 문서의 첫 번째 항목은 *정확하게 우리가 작성하려고 한 것이고, 그것이 왜 작동하지 않았는지, 어떻게 해결해야 하는지* 를 알려줍니다.

젠장, 문서가 지배합니다.

좋아요, 그렇게 하죠:

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

완성되었습니다!

...하지만 이것은 사실 몇 가지 이유로 인해 리스트에 대한 정말 어리석은 정의입니다.

두 개의 요소가 있는 리스트를 생각해 봅시다:

```text
[] = Stack
() = Heap

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *junk*)
```

두 가지 중요한 문제가 있습니다:

* "나는 실제로 노드가 아니다"라고만 말하는 노드를 할당하고 있습니다.
* 노드 중 하나가 힙 할당이 전혀 되지 않습니다.

표면적으로는 이 두 가지가 서로 상쇄되는 것처럼 보입니다. 여분의 노드를 힙 할당하지만 노드 중 하나는 힙 할당할 필요가 전혀 없기 때문입니다. 하지만 다음과 같은 리스트 레이아웃을 고려해 보세요:

```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

이 레이아웃에서는 이제 무조건 노드를 힙에 할당합니다. 가장 큰 차이점은 첫 번째 레이아웃에 있던 *정크*가 없다는 것입니다. 이 정크는 무엇일까요? 이를 이해하려면 열거형이 메모리에 어떻게 배치되는지 살펴볼 필요가 있습니다.

일반적으로 다음과 같은 열거 형이 있으면:

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

Foo는 자신이 나타내는 열거형의 *변형*(`D1`, `D2`, ... `Dn`)을 나타내는 정수를 저장해야 합니다. 이것이 열거형의 *태그*입니다. 또한 `T1`, `T2`, ... `Tn` 중 *가장 큰 것*을 저장하기에 충분한 공간이 필요합니다(정렬 요건을 충족하기 위해 약간의 추가 공간도 필요).

여기서 중요한 점은 `Empty`는 단일 비트 정보이지만 언제든지 `Elem`이 될 준비가 되어 있어야 하기 때문에 포인터와 요소를 위한 충분한 공간을 반드시 소비한다는 것입니다. 따라서 첫 번째 레이아웃 힙은 정크로 가득 찬 추가 요소를 할당하여 두 번째 레이아웃보다 조금 더 많은 공간을 소비합니다.

노드 중 하나를 전혀 할당하지 않는 것도 놀랍게도 항상 할당하는 것보다 *더 나쁩니다*. 이는 노드 레이아웃이 *비균일*해지기 때문입니다. 이는 노드를 푸시하고 팝하는 데는 별다른 영향을 미치지 않지만 목록을 분할하고 병합하는 데는 영향을 미칩니다.

두 레이아웃에서 목록을 분할하는 것을 고려해 보세요:

```text
layout 1:

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

split off C:

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
layout 2:

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

split off C:

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

레이아웃 2의 분할은 B의 포인터를 스택으로 복사하고 이전 값을 무효화하기만 하면 됩니다. 레이아웃 1은 궁극적으로 동일한 작업을 수행하지만, 힙에서 스택으로 C를 복사해야 합니다. 병합은 그 반대로 같은 과정을 거칩니다.

연결 리스트의 몇 안 되는 좋은 점 중 하나는 노드 자체에서 요소를 구성한 다음, 요소를 이동하지 않고도 목록에서 자유롭게 섞을 수 있다는 것입니다. 포인터를 조작하기만 하면 요소가 '이동'됩니다. 레이아웃 1은 이 속성을 버립니다.

좋아, 레이아웃 1이 나쁘다고 확신합니다. 목록을 어떻게 다시 작성할까요? 다음과 같이 할 수 있습니다:

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

여러분에게는 더 나쁜 생각처럼 보일 수도 있습니다. 가장 주목할 만한 점은 이제 완전히 유효하지 않은 상태가 발생하기 때문에 논리가 매우 복잡해진다는 것입니다: `ElemThenNotEmpty(0, Box(Empty))`입니다. 또한 요소를 균일하게 할당하지 않는 문제도 *여전히* 있습니다.

하지만 *한 가지* 흥미로운 속성이 있습니다: Empty 케이스의 할당을 완전히 피하여 총 힙 할당 횟수를 1만큼 줄입니다. 안타깝게도 그렇게 함으로써 *더 많은* 공간을 낭비하게 됩니다! 이는 이전 레이아웃이 *널 포인터 최적화*를 활용했기 때문입니다.

앞서 모든 열거형에는 해당 비트가 나타내는 열거형의 변형을 지정하는 *태그*가 저장되어야 한다는 것을 살펴봤습니다. 하지만 특별한 종류의 열거형이 있다면:

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

이면 널 포인터 최적화가 시작되어 *태그에 필요한 공간을 제거*합니다. 변형이 A인 경우 전체 열거형은 모두 `0`으로 설정됩니다. 그렇지 않으면 변형이 B인 경우 0이 아닌 포인터가 포함되어 있기 때문에 B가 모두 `0`이 될 수 없기 때문에 작동합니다. 멋지네요!

이런 종류의 최적화를 수행할 수 있는 다른 열거형과 타입을 생각해 볼 수 있나요? 사실 아주 많습니다! 이것이 바로 Rust가 열거형 레이아웃을 완전히 지정하지 않은 채로 두는 이유입니다. Rust가 제공하는 몇 가지 복잡한 열거형 레이아웃 최적화가 더 있지만, 그중에서도 널 포인터가 가장 중요합니다! 이는 `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec` 등 Rust에서 중요한 여러 타입을 `Option`에 넣을 때 오버헤드가 없음을 의미합니다! (이 중 대부분은 조만간 설명하겠습니다).

그렇다면 어떻게 하면 여분의 정크를 피하고, 균일하게 할당하고, *그리고* 그 달콤한 널 포인터 최적화를 얻을 수 있을까요? 요소를 갖는다는 생각과 다른 목록을 할당한다는 생각을 더 잘 구분해야 합니다. 이를 위해서는 조금 더 C처럼 생각해야 합니다: 구조체!

열거형은 여러 값 중 *하나*를 포함할 수 있는 유형을 선언할 수 있는 반면, 구조체는 한 번에 *여러* 값을 포함하는 유형을 선언할 수 있습니다. List를 두 가지 타입으로 나눠보겠습니다: 리스트와 노드입니다.

이전과 마찬가지로, 리스트는 비어 있거나 요소가 있고 그 뒤에 다른 리스트가 있습니다. "요소가 있고 그 뒤에 다른 List가 있는 경우"를 완전히 별도의 타입으로 표현하면 Box를 보다 최적의 위치에 올릴 수 있습니다:

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```
우선순위를 확인해 봅시다:

* 리스트의 꼬리는 여분의 정크를 할당하지 않습니다: 체크!
* `열거형`은 널 포인터에 최적화되어 있음: 체크!
* 모든 요소가 균일하게 할당됨: 체크!

끝났습니다! 실제로 첫 번째 레이아웃(공식 Rust 문서에서 제안한 대로)에 문제가 있다는 것을 보여주기 위해 사용한 레이아웃을 정확히 구성했습니다.

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(
Rust가 또 화를 냈습니다. 우리는 `List`를 공개로 표시했지만(사람들이 사용할 수 있기를 원하기 때문에), `Node`는 공개로 표시하지 않았습니다. 문제는 `열거형`의 내부가 완전히 공개되어 있고 비공개 유형에 대해 공개적으로 이야기할 수 없다는 것입니다. 모든 `Node`를 완전히 공개할 수도 있지만, 일반적으로 Rust에서는 구현 세부 사항을 비공개로 유지하는 것을 선호합니다. 구현 세부 사항을 숨길 수 있도록 `List`를 구조체로 만들어 봅시다:

```rust ,ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

`List`는 단일 필드가 있는 구조체이므로 그 크기는 해당 필드와 동일합니다. 비용이 들지 않는 추상화!


```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```
좋아, 컴파일이 완료되었습니다! 왜냐하면 우리가 작성한 모든 것이 완전히 쓸모없기 때문입니다. 우리는 `head`를 사용하지 않으며 라이브러리를 사용하는 누구도 비공개이므로 사용할 수 없기 때문입니다. 이는 곧 링크와 노드도 쓸모없다는 뜻입니다. 그럼 이 문제를 해결해 봅시다! List에 대한 코드를 구현해 봅시다!
