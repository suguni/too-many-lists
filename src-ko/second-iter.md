# Iter

자, 이제 Iter를 구현해봅시다. 이번에는 List가 우리가 원하는 모든 기능을 제공하지 않기 때문에, 자체적으로 구현해야 합니다. 우리가 원하는 기본 로직은 다음에 반환할 노드를 가리키는 포인터를 유지하는 것입니다. 그 노드가 존재하지 않을 수 있으므로(리스트가 비어 있거나 반복이 끝난 경우), 그 참조를 Option으로 사용하고자 합니다. 요소를 반환할 때는 현재 노드의 `next` 노드로 진행하고자 합니다.

자, 시작해봅시다:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

오 이런. 라이프타임입니다. 이것들은 들어본 적 있습니다. 악몽이라고 들었습니다.

새로운 것을 시도해봅시다: `error[E0106]` 오류 코드 보이시나요? 이는 컴파일러 오류 코드입니다. 우리는 `--explain`을 통해 rustc에 이 오류를 설명해달라고 요청할 수 있습니다:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

이 설명은 많이 명확하지 않네요(이 문서는 우리가 Rust를 더 잘 이해하고 있다고 가정합니다). 하지만 구조체에 `'a`를 추가해야 할 것 같습니다. 시도해봅시다.

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

패턴을 알 것 같네요... 이 작은 것을 모든 곳에 추가해봅시다:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

오 이런. Rust를 망가뜨렸네요.

이 `'a` 라이프타임이 무엇을 의미하는지 실제로 이해해야 할 것 같습니다.

라이프타임은 많은 사람들을 겁먹게 할 수 있습니다. 우리가 프로그래밍을 처음 시작할 때부터 알고 사랑해온 것들에 대한 변화이기 때문입니다. 사실 지금까지 우리는 라이프타임을 피할 수 있었지만, 프로그램 전반에 걸쳐 얽혀 있었습니다.

가비지 컬렉션 언어에서는 가비지 컬렉터가 모든 것을 마법처럼 필요한 만큼만 유지시켜주기 때문에 라이프타임이 필요하지 않습니다. Rust의 대부분의 데이터는 *수동으로* 관리되므로, 다른 해결책이 필요합니다. C와 C++은 사람들이 스택의 임의 데이터에 대한 포인터를 갖도록 허용할 때 발생하는 문제를 명확하게 보여줍니다: 광범위한 관리 불가능한 불안전성. 이는 대략 두 가지 오류로 나눌 수 있습니다:

* 스코프를 벗어난 것을 가리키는 포인터를 보유
* 변경된 것을 가리키는 포인터를 보유

라이프타임은 이 두 가지 문제를 해결하며, 99%의 경우 이는 완전히 투명한 방식으로 수행됩니다.

그렇다면 라이프타임이란 무엇일까요?

간단히 말해서, 라이프타임은 프로그램 내의 코드 영역(대략 블록/스코프)의 이름입니다. 그게 전부입니다. 참조에 라이프타임이 태그되면, 그 참조는 해당 *전체* 영역 동안 유효해야 한다고 말하는 것입니다. 다양한 요소들이 참조가 유효해야 하는 기간과 유효할 수 있는 기간에 대한 요구 사항을 부여합니다. 전체 라이프타임 시스템은 이러한 요구 사항을 최소화하려는 제약 조건 해결 시스템일 뿐입니다. 모든 제약 조건을 충족하는 라이프타임 세트를 성공적으로 찾으면 프로그램이 컴파일됩니다! 그렇지 않으면 무언가가 충분히 오래 살지 못했다고 말하는 오류를 받게 됩니다.

함수 본문 내에서는 일반적으로 라이프타임에 대해 이야기할 수 없으며, *어쨌든* 그렇게 하고 싶지 않을 것입니다. 컴파일러는 모든 정보를 가지고 있으며 모든 제약 조건을 유추하여 최소 라이프타임을 찾을 수 있습니다. 그러나 타입 및 API 수준에서는 컴파일러가 모든 정보를 가지고 있지 *않습니다*. 서로 다른 라이프타임 간의 관계를 알려주어야 컴파일러가 무엇을 하고 있는지 파악할 수 있습니다.

원칙적으로, 이러한 라이프타임을 생략할 *수도* 있지만, 모든 차용을 확인하는 것은 전체 프로그램 분석이 되어 비지역적인 오류를 초래할 수 있습니다. Rust의 시스템 덕분에 모든 차용 검사는 각 함수 본문에서 독립적으로 수행될 수 있으며, 모든 오류는 비교적 지역적으로 발생합니다(또는 타입 서명이 잘못되었을 수 있습니다).

하지만 우리는 이미 함수 서명에서 참조를 작성했으며, 이는 문제가 되지 않았습니다! 이는 특정 경우가 너무 일반적이어서 Rust가 자동으로 라이프타임을 선택하기 때문입니다. 이것이 *라이프타임 생략*입니다.

특히:

```rust ,ignore
// 입력에 하나의 참조만 있는 경우, 출력은 해당 입력에서 유도됩니다.
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// 여러 입력, 모두 독립적이라고 가정
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// 메서드, 모든 출력 라이프타임은 `self`에서 유도된다고 가정
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

그렇다면 `fn foo<'a>(&'a A) -> &'a B`의 *의미*는 무엇일까요? 실용적으로는 입력이 출력만큼 오래 살아야 한다는 것을 의미합니다. 출력이 오랫동안 유지된다면 입력도 그만큼 오랫동안 유효해야 하며, 출력을 사용하지 않으면 입력이 무효화될 수 있다는 것을 컴파일러가 알게 됩니다.

이 시스템 덕분에 Rust는 해제 후 사용을 방지하고, 참조가 존재하는 동안 데이터가 변경되지 않음을 보장합니다.

자, Iter로 돌아가서 라이프타임이 없는 상태로 되돌려 봅시다:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

우리는 함수 및 타입 서명에만 라이프타임을 추가해야 합니다:

```rust ,ignore
// Iter는 *어떤* 라이프타임에 대해 제네릭하며, 신경 쓰지 않습니다
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// 여기서는 라이프타임이 없습니다, List에는 관련된 라이프타임이 없습니다
impl<T> List<T> {
    // 여기서 *정확한* 차용을 위해 새로운 라이프타임을 선언합니다
    // iter가 생성되는 차용입니다. 이제 &self는 Iter가 유지되는 동안 유효해야 합니다
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// Iter에는 라이프타임이 있으므로 정의*해야* 합니다
impl<'a, T> Iterator for Iter<'a, T> {
    // 여기서도 필요합니다, 이는 타입 선언입니다
    type Item = &'a T;

    // 이 부분은 변경할 필요가 없으며, 위에서 처리됩니다
    // Self는 여전히 매우 놀랍고 놀라운 기능입니다
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

좋습니다, 이번에는 제대로 된 것 같습니다.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

OK. SO. 우리는 라이프타임 오류를 수정했지만, 이제 새로운 타입 오류가 발생하고 있습니다.

우리는 `&Node`를 저장하고 싶지만, `&Box<Node>`를 얻고 있습니다. 상자를 역참조한 후 참조를 가져오면 쉽게 해결할 수 있습니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

`as_ref`를 잊었습니다. 그래서 상자를 `map`으로 이동시키고 있습니다. 이는 상자가 제거된다는 의미이며, 참조가 댕글링하게 됩니다:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref`가 추가적인 간접 참조 계층을 추가했습니다. 이를 제거해야 합니다:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```text
cargo build

```

🎉 🎉 🎉

as\_deref 및 as\_deref_mut 함수는 Rust 1.40부터 안정적입니다. 그 전에는 `map(|node| &**node)` 및 `map(|node| &mut**node)`를 사용해야 했습니다. "이 `&**`가 정말 이상하다"고 생각할 수 있으며, 틀리지 않았습니다. 그러나 좋은 와인처럼 Rust는 시간이 지남에 따라 개선되어 더 이상 이런 작업을 할 필요가 없습니다. 일반적으로 Rust는 *deref 강제 변환*이라는 과정을 통해 이러한 변환을 암시적으로 잘 수행합니다. 이 과정에서는 타입 검사를 위해 코드에 \*를 삽입할 수 있습니다. 이는 차용 검사기가 포인터를 망치지 않도록 보장하기 때문에 가능합니다!

하지만 이 경우, 클로저와 `Option<&T>`가 `&T`` 대신 결합되었기 때문에 이는 너무 복잡하여 Rust가 이를 해결할 수 없습니다. 따라서 명시적으로 이를 도와야 합니다. 다행히도 이는 드물게 발생합니다.

완전성을 위해 *turbofish*를 사용하여 *다른* 힌트를 *줄 수*도 있습니다:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

map은 제네릭 함수입니다:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

turbofish(`::<>`)는 제네릭 타입을 명시할 수 있게 해줍니다. 이 경우 `::<&Node<T>, _>`는 "`&Node<T>`를 반환해야 한다"는 것을 의미합니다.

이렇게 하면 `&node`에 역참조 강제 변환이 적용되어, \*를 수동으로 적용할 필요가 없습니다. 하지만 이 경우 실질적인 개선은 아닙니다.

테스트를 작성하여 우리가 아무 것도 하지 않았는지 확인해봅시다:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

좋습니다.

마지막으로, 실제로 라이프타임 생략을 적용*할 수* 있습니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

is equivalent to:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

적은 라이프타임!

또는 구조체에 라이프타임이 포함된다는 것을 "숨기기" 싫다면, Rust 2018 "명시적으로 생략된 라이프타임" 문법인 `'_`을 사용할 수 있습니다:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```
