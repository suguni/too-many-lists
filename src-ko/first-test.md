# Testing
이제 `push`와 `pop`이 작성되었으니 실제로 스택을 테스트해 볼 수 있습니다! Rust와 카고는 일급 기능으로 테스트를 지원하므로 매우 쉽습니다. 함수를 작성하고 `#[test]`로 주석을 달기만 하면 됩니다.

일반적으로 Rust 커뮤니티에서 테스트 중인 코드 옆에 테스트를 유지하려고 합니다. 그러나 "실제" 코드와 충돌을 피하기 위해 일반적으로 테스트에 대한 새로운 네임스페이스를 만듭니다. `mod`를 사용하여 `first.rs`가 `lib.rs`에 포함되도록 지정한 것처럼, `mod`를 사용하여 기본적으로 완전히 새로운 파일을 *인라인*으로 만들 수 있습니다:

```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```
그리고 `cargo test`로 호출합니다.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

아무것도 하지 않기 테스트가 통과되었습니다! 이제 아무것도 하지 않기가 아닌 테스트를 만들어 봅시다. `assert_eq!` 매크로로 이를 수행하겠습니다. 이것은 특별한 테스트 마법이 아닙니다. 그저 두 가지를 비교해서 일치하지 않으면 프로그램을 panic 하게 만들면 됩니다. 네, 테스트 하네스를 당황하게 함으로써 실패를 나타냅니다!

```rust ,ignore
mod test {
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

죄송합니다! 새 모듈을 만들었으므로 이를 사용하려면 명시적으로 List를 가져와야 합니다.

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

야호!

그런데 왜 저런 경고가 뜨는 건가요...? 저희는 분명히 테스트에서 리스트를 사용합니다!

...하지만 테스트할 때만! 컴파일러를 달래기 위해(그리고 소비자에게 친근감을 주기 위해) 테스트를 실행하는 경우에만 전체 `test` 모듈을 컴파일해야 한다는 것을 표시해야 합니다.


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

여기까지가 테스트의 모든 것입니다!
