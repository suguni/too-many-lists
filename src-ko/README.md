# Learn Rust With Entirely Too Many Linked Lists

> Got any issues or want to check out all the final code at once?
> [Everything's on Github!][github]

> **NOTE**: The current edition of this book is written against Rust 2018,
> which was first released with rustc 1.31 (Dec 8, 2018). If your rust toolchain
> is new enough, the Cargo.toml file that `cargo new` creates should contain the
> line `edition = "2018"` (or if you're reading this in the far future, perhaps
> some even larger number!). Using an older toolchain is possible, but unlocks
> a secret **hardmode**, where you get extra compiler errors that go completely
> unmentioned in the text of this book. Wow, sounds like fun!

Rust에서 연결 리스트(linked list)을 구현하는 방법에 대한 질문을 꽤 자주 받습니다. 솔직히 말해서 대답은 요구사항이 무엇인지에 따라 다르며, 그 자리에서 대답하기는 매우 쉽지 않습니다. 그래서 저는 이 책을 집필하여 이 질문에 대한 포괄적인 답변을 드리기로 결심했습니다.

I fairly frequently get asked how to implement a linked list in Rust. The answer honestly depends on what your requirements are, and it's obviously not super easy to answer the question on the spot. As such I've decided to write this book to comprehensively answer the question once and for all.

이 시리즈에서는 6개의 연결 리스트를 구현하는 것을 통해 기본 및 고급 Rust 프로그래밍을 알려드리겠습니다. 이를 통해 여러분은 다음 내용들을 배우게 될 것입니다:

In this series I will teach you basic and advanced Rust programming entirely by having you implement 6 linked lists. In doing so, you should learn:

* 다음의 포인터 타입: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`(?)
* 소유권(Ownership), 빌림(borrowing), inherited mutability, interior mutability, Copy
* 모든 키워드들 : struct, enum, fn, pub, impl, use, ...
* 패턴 매칭, 제너릭, 디스트럭쳐
* 테스팅, `miri` 를 이용해 새로운 톨체인 설치하기
* Unsafe Rust: raw pointers, aliasing, stacked borrows, UnsafeCell, variance

* The following pointer types: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`(?)
* Ownership, borrowing, inherited mutability, interior mutability, Copy
* All The Keywords: struct, enum, fn, pub, impl, use, ...
* Pattern matching, generics, destructors
* Testing, installing new toolchains, using `miri`
* Unsafe Rust: raw pointers, aliasing, stacked borrows, UnsafeCell, variance

네, 연결리스트는 정말 끔찍해서 그들을 현실로 만드려면 이 모든 개념들을 다루어야 합니다.

Yes, linked lists are so truly awful that you deal with all of these concepts in making them real.

모든 내용은 사이드바에 있지만(모바일에서는 축소되어 있을 수 있음), 간단히 참고할 수 있도록 만들려는 내용은 다음과 같습니다:

Everything's in the sidebar (may be collapsed on mobile), but for quick reference, here's what we're going to be making:

1. [A Bad Singly-Linked Stack](first.md)
2. [An Ok Singly-Linked Stack](second.md)
3. [A Persistent Singly-Linked Stack](third.md)
4. [A Bad But Safe Doubly-Linked Deque](fourth.md)
5. [An Unsafe Singly-Linked Queue](fifth.md)
6. [TODO: An Ok Unsafe Doubly-Linked Deque](sixth.md)
7. [Bonus: A Bunch of Silly Lists](infinity.md)

모두 같은 맥락에서 이해할 수 있도록 터미널에 입력하는 모든 명령어를 적어보겠습니다. 또한 Rust의 표준 패키지 관리자인 Cargo를 사용해 프로젝트를 개발할 것입니다. Cargo는 Rust 프로그램을 작성하는 데 꼭 필요한 것은 아니지만, rustc를 직접 사용하는 것보다 훨씬 *더* 좋습니다. 그냥 주위를 둘러보고 싶다면 [play.rust-lang.org][play]를 통해 브라우저에서 몇 가지 간단한 프로그램을 실행할 수도 있습니다.

Just so we're all the same page, I'll be writing out all the commands that I feed into my terminal. I'll also be using Rust's standard package manager, Cargo, to develop the project. Cargo isn't necessary to write a Rust program, but it's *so much* better than using rustc directly. If you just want to futz around you can also run some simple programs in the browser via [play.rust-lang.org][play].

이후 섹션에서는 "rustup"을 사용해 추가 Rust 도구들을 설치하겠습니다. rustup을 사용하여 [모든 Rust 툴체인을 설치하는 것](https://www.rust-lang.org/tools/install)을 강력히 권장합니다.

In later sections, we'll be using "rustup" to install extra Rust tooling. I strongly recommend [installing all of your Rust toolchains using rustup](https://www.rust-lang.org/tools/install).

이제 프로젝트를 시작해 보겠습니다:

Let's get started and make our project:

```text
> cargo new --lib lists
> cd lists
```

각 리스트를 별도의 파일에 넣어 작업 내용을 잃어버리지 않도록 하겠습니다.

We'll put each list in a separate file so that we don't lose any of our work.

Rust를 *제대로* 배우는 경험은 코드를 작성하고 컴파일러가 소리를 지르며 도대체 무슨 뜻인지 알아내려고 노력하는 경험들을 수반해야 한다는 점에 유의하세요. 가능한 한 자주 이런 일이 발생하지 않도록 주의하겠습니다. 일반적으로 우수한 컴파일러 오류와 문서를 읽고 이해하는 방법을 배우는 것은 생산적인 Rust 프로그래머가 되기 위해 *무엇보다* 중요합니다.

It should be noted that the *authentic* Rust learning experience involves writing code, having the compiler scream at you, and trying to figure out what the heck that means. I will be carefully ensuring that this occurs as frequently as possible. Learning to read and understand Rust's generally excellent compiler errors and documentation is *incredibly* important to being a productive Rust programmer.

사실 그건 거짓말입니다. 이 글을 쓰면서 제가 보여드리는 것보다 훨씬 더 많은 컴파일러 오류가 발생했습니다. 특히 이후 장에서는 모든 언어에서 발생할 수 있는 "잘못 입력(복사 붙여넣기)했습니다"라는 오류는 많이 보여드리지 않겠습니다. 이것은 컴파일러가 우리에게 비명을 지르게 하는 *가이드 투어*입니다.

Although actually that's a lie. In writing this I encountered *way* more compiler errors than I show. In particular, in the later chapters I won't be showing a lot of the random "I typed (copy-pasted) bad" errors that you expect to encounter in every language. This is a *guided tour* of having the compiler scream at us.

우리는 꽤 느리게 진행할 것이고, 저는 솔직히 거의 내내 진지하지 않을 것입니다. 프로그래밍은 재미있어야 한다고 생각해요, 젠장! 최대한 정보 밀도가 높고 진지하고 형식적인 콘텐츠를 원하는 사람이라면 이 책은 적합하지 않습니다. 제가 만드는 어떤 것도 여러분을 위한 것이 아닙니다. 틀린 생각입니다.

We're going to be going pretty slow, and I'm honestly not going to be very serious pretty much the entire time. I think programming should be fun, dang it! If you're the type of person who wants maximally information-dense, serious, and formal content, this book is not for you. Nothing I will ever make is for you. You are wrong.




# 의무 공익 광고 An Obligatory Public Service Announcement

확실히 말씀드리자면, 저는 연결 리스트를 싫어합니다. 열정적으로요. 연결 리스트는 끔찍한 데이터 구조입니다. 물론 연결 리스트에 대한 몇 가지 훌륭한 사용 사례가 있습니다:

Just so we're totally 100% clear: I hate linked lists. With a passion. Linked lists are terrible data structures. Now of course there's several great use cases for a linked list:

* 큰 목록을 분할하거나 병합하는 작업을 *많이* 하고 싶습니다. * 많이*.
* 잠금 없는(lock-free) 멋진 동시 작업을 하고 있는 경우.
* 커널/임베디드 무언가를 작성하고 있고 침입형(intrusive) 목록을 사용하고 싶은 경우.
* 순수한 함수형 언어를 사용하고 있고, 제한된 의미론과 변화의 부재(불변)이 연결 리스트로 작업하기 더 쉽게 해 주는 경우.
* ... 그리고 더!

* You want to do *a lot* of splitting or merging of big lists. *A lot*.
* You're doing some awesome lock-free concurrent thing.
* You're writing a kernel/embedded thing and want to use an intrusive list.
* You're using a pure functional language and the limited semantics and absence of mutation makes linked lists easier to work with.
* ... and more!

하지만 이 모든 경우는 Rust 프로그램을 작성하는 사람에게는 *매우 드문 경우*입니다. 99%의 경우 Vec(배열 스택)을 사용해야 하고, 나머지 1%의 경우 VecDeque(배열 데크)를 사용해야 합니다. 이는 할당 빈도가 낮고, 메모리 오버헤드가 적으며, 진정한 랜덤 액세스 및 캐시 로컬리티를 제공하므로 대부분의 워크로드에서 명백히 우수한 데이터 구조입니다.

But all of these cases are *super rare* for anyone writing a Rust program. 99% of the time you should just use a Vec (array stack), and 99% of the other 1% of the time you should be using a VecDeque (array deque). These are blatantly superior data structures for most workloads due to less frequent allocation, lower memory overhead, true random access, and cache locality.


연결 리스트는 트라이만큼이나 * 틈새적이고 * 모호한* 데이터 구조입니다. 트라이가 일반 프로그래머가 평생 배우기 힘든 틈새 구조라고 주장하는 저에게 반박할 사람은 거의 없지만, 링크드 리스트는 기이한 유명인사의 지위를 가지고 있습니다. 저희는 모든 학부생에게 링크드 리스트 작성법을 가르칩니다. 이것은 [제가 std::collection에서 죽일 수 없었던][rust-std-list] 유일한 틈새 컬렉션입니다. [C++에 있는 바로 그 리스트][cpp-std-list]입니다!

Linked lists are as *niche* and *vague* of a data structure as a trie. Few would balk at me claiming a trie is a niche structure that your average programmer could happily never learn in an entire productive career -- and yet linked lists have some bizarre celebrity status. We teach every undergrad how to write a linked list. It's the only niche collection [I couldn't kill from std::collections][rust-std-list]. It's [*the* list in C++][cpp-std-list]!

우리 커뮤니티는 모두 연결 리스트를 "표준" 데이터 구조로 사용하는 것에 대해 *아니오*라고 말해야 합니다. 몇 가지 훌륭한 사용 사례가 있는 훌륭한 데이터 구조이지만, 이러한 사용 사례는 일반적인 것이 아니라 *예외적인* 경우입니다.

We should all as a community say *no* to linked lists as a "standard" data structure. It's a fine data structure with several great use cases, but those use cases are *exceptional*, not common.

이 PSA의 첫 문단만 읽은 후 읽기를 중단하는 사람들이 많습니다. 말 그대로 *훌륭한 사용 사례* 목록에 있는 것들 중 하나를 나열하며 제 주장을 반박하려고 하죠. 첫 문단 바로 뒤에 나오는 것!

Several people apparently read the first paragraph of this PSA and then stop reading. Like, literally they'll try to rebut my argument by listing one of the things in my list of *great use cases*. The thing right after the first paragraph!

자세한 논증으로 바로 연결할 수 있도록 제가 본 몇 가지 반론과 그에 대한 저의 답변을 소개합니다. Rust에 대해 조금만 배우고 싶으시면 [첫 번째 장](first.md)으로 건너뛰셔도 됩니다!

Just so I can link directly to a detailed argument, here are several attempts at counter-arguments I have seen, and my response to them. Feel free to skip to [the first chapter](first.md) if you just want to learn some Rust!




## 성능이 항상 중요한 것은 아닙니다. - Performance doesn't always matter

예! 애플리케이션이 I/O에 바인딩되어 있거나 문제의 코드가 중요하지 않은 콜드 케이스에 있을 수도 있습니다. 하지만 이것은 연결 리스트를 사용하기 위한 인수가 아닙니다. 이것은 *무엇이든* 사용해야 한다는 주장입니다. 왜 연결 리스트에 안주할까요? 연결 해시 맵을 사용하세요!

Yes! Maybe your application is I/O-bound or the code in question is in some cold case that just doesn't matter. But this isn't even an argument for using a linked list. This is an argument for using *whatever at all*. Why settle for a linked list? Use a linked hash map!

성능이 중요하지 않다면, *당연히* 자연스러운 기본값인 배열을 적용해도 괜찮습니다.

If performance doesn't matter, then it's *surely* fine to apply the natural default of an array.





## 포인터가 있는 경우 O(1) 분할-추가-삽입-제거가 됩니다. - They have O(1) split-append-insert-remove if you have a pointer there

맞습니다! [비야르네 스트로스트럽이 지적][bjarne]했듯이, 포인터를 가져오는 데 걸리는 시간이 배열의 모든 요소를 복사하는 데 걸리는 시간(실제로는 매우 빠릅니다)보다 완전히 줄어든다면 *실제로는 중요하지 않습니다*.

Yep! Although as [Bjarne Stroustrup notes][bjarne] *this doesn't actually matter* if the time it takes to get that pointer completely dwarfs the time it would take to just copy over all the elements in an array (which is really quite fast).

분할 및 병합 비용이 크게 좌우하는 워크로드가 아니라면 캐싱 효과와 코드 복잡성으로 인해 *다른 모든* 작업이 받는 페널티로 인해 이론적인 이득은 사라지게 됩니다.

Unless you have a workload that is heavily dominated by splitting and merging costs, the penalty *every other* operation takes due to caching effects and code complexity will eliminate any theoretical gains.

*하지만 애플리케이션을 프로파일링하여 분할 및 병합에 많은 시간을 할애하는 경우 연결된 목록에서 이득을 얻을 수 있습니다*.

*But yes, if you're profiling your application to spend a lot of time in splitting and merging, you may have gains in a linked list*.





## 할부 상환을 감당할 수 없습니다. - I can't afford amortization

대부분 할부 상환을 감당할 수 있는 틈새 시장에 이미 진입한 것입니다. 하지만 배열은 *최악의 경우* 상각됩니다. 배열을 사용한다고 해서 상각 비용이 발생하는 것은 아닙니다. 얼마나 많은 요소를 저장할지 예측할 수 있다면(또는 상한선이 있다면) 필요한 모든 공간을 미리 예약할 수 있습니다. 제 경험상 얼마나 많은 요소가 필요한지 예측할 수 있는 것은 매우 흔한 일입니다. 특히 Rust에서는 모든 이터레이터가 정확히 이 경우에 대한 `size_hint`를 제공합니다.

You've already entered a pretty niche space -- most can afford amortization. Still, arrays are amortized *in the worst case*. Just because you're using an array, doesn't mean you have amortized costs. If you can predict how many elements you're going to store (or even have an upper-bound), you can pre-reserve all the space you need. In my experience it's *very* common to be able to predict how many elements you'll need. In Rust in particular, all iterators provide a `size_hint` for exactly this case.

그러면 `push`와 `pop`은 진정한 O(1) 연산이 됩니다. 그리고 연결 리스트에서 `push`와 `pop`보다 *상당히* 빠를 것입니다. 포인터 오프셋을 하고, 바이트를 쓰고, 정수를 증가시키면 됩니다. 어떤 종류의 얼로케이터도 사용할 필요가 없습니다.

Then `push` and `pop` will be truly O(1) operations. And they're going to be *considerably* faster than `push` and `pop` on linked list. You do a pointer offset, write the bytes, and increment an integer. No need to go to any kind of allocator.

지연 시간이 짧지 않나요?

How's that for low latency?

*예, 하지만 부하를 예측할 수 없다면 최악의 경우 지연 시간을 줄일 수 있습니다!

*But yes, if you can't predict your load, there are worst-case latency savings to be had!*





## 링크된 목록은 공간을 덜 낭비합니다. - Linked lists waste less space

글쎄요, 이것은 복잡합니다. "표준" 배열 크기 조정 전략은 배열의 최대 절반이 비어 있도록 늘리거나 줄이는 것입니다. 이것은 실제로 많은 공간을 낭비하는 것입니다. 특히 Rust에서는 컬렉션을 자동으로 축소하지 않기 때문에(다시 채우려고 하면 낭비입니다) 낭비되는 공간이 무한대에 가까워질 수 있습니다!

Well, this is complicated. A "standard" array resizing strategy is to grow or shrink so that at most half the array is empty. This is indeed a lot of wasted space. Especially in Rust, we don't automatically shrink collections (it's a waste if you're just going to fill it back up again), so the wastage can approach infinity!

하지만 이것은 최악의 시나리오입니다. 최상의 경우 배열 스택에는 전체 배열에 대해 3개의 포인터만 오버헤드가 발생합니다. 기본적으로 오버헤드가 없습니다.

But this is a worst-case scenario. In the best-case, an array stack only has three pointers of overhead for the entire array. Basically no overhead.

반면에 연결된 목록은 요소당 무조건 공간을 낭비합니다. 단일 링크된 목록은 하나의 포인터를 낭비하고 이중 링크된 목록은 두 개의 포인터를 낭비합니다. 배열과 달리 상대적인 낭비는 요소의 크기에 비례합니다. 요소가 *거대한* 경우 낭비가 0에 가까워집니다. 작은 요소(예: 바이트)가 있는 경우 메모리 오버헤드가 최대 16배(32비트에서는 8배)가 될 수 있습니다!

Linked lists on the other hand unconditionally waste space per element. A singly-linked list wastes one pointer while a doubly-linked list wastes two. Unlike an array, the relative wasteage is proportional to the size of the element. If you have *huge* elements this approaches 0 waste. If you have tiny elements (say, bytes), then this can be as much as 16x memory overhead (8x on 32-bit)!

실제로는 전체 노드의 크기를 포인터에 맞추기 위해 바이트에 패딩이 추가되기 때문에 23배(32비트에서는 11배)에 가깝습니다.

Actually, it's more like 23x (11x on 32-bit) because padding will be added to the byte to align the whole node's size to a pointer.

이는 또한 노드 할당 및 할당 해제가 조밀하게 이루어지고 조각화로 인해 메모리가 손실되지 않는다는 할당자의 최상의 경우를 가정한 것입니다.

This is also assuming the best-case for your allocator: that allocating and deallocating nodes is being done densely and you're not losing memory to fragmentation.

*하지만 요소가 방대하고 부하를 예측할 수 없으며 적절한 할당자가 있다면 메모리를 절약할 수 있습니다!

*But yes, if you have huge elements, can't predict your load, and have a decent allocator, there are memory savings to be had!*





## 저는 &lt;함수형 언어&gt;에서 항상 연결 리스트를 사용합니다 - I use linked lists all the time in &lt;functional language&gt;

훌륭하죠! 연결된 목록은 변이 없이 조작할 수 있고 재귀적으로 설명할 수 있으며 게으름의 마법으로 인해 무한한 목록으로 작업할 수도 있기 때문에 함수형 언어에서 사용하기에 매우 우아합니다.

Great! Linked lists are super elegant to use in functional languages because you can manipulate them without any mutation, can describe them recursively, and also work with infinite lists due to the magic of laziness.

특히 링크된 목록은 변경 가능한 상태가 필요 없는 반복을 나타내므로 유용합니다. 다음 단계는 다음 하위 목록을 방문하는 것입니다.

Specifically, linked lists are nice because they represent an iteration without the need for any mutable state. The next step is just visiting the next sublist.

Rust는 대부분 [이터레이터][]로 이런 종류의 작업을 수행합니다. 무한대로 만들 수 있으며 함수형 리스트처럼 매핑, 필터링, 반전, 연결할 수 있으며 이 모든 것이 게으르게(lazily) 수행됩니다!

Rust mostly does this kind of thing with [iterators][]. They can be infinite  and you can map, filter, reverse, and concatenate them just like a functional list, and it will all be done just as lazily!

Rust에서는 *[슬라이스][]*로 하위 배열에 대해서도 쉽게 이야기할 수 있습니다. 함수형 언어에서 일반적인 헤드/테일 분할은 [그냥 `slice.split_at_mut(1)`][split]입니다. 오랫동안 Rust에는 매우 멋진 슬라이스 패턴 매칭을 위한 실험적인 시스템이 있었지만, 안정화되면서 기능이 단순화되었습니다. 그래도 [기본 슬라이스 패턴][slice-pats]은 깔끔합니다! 물론 슬라이스를 이터레이터로 전환할 수도 있습니다!

Rust also lets you easily talk about sub-arrays with *[slices][]*. Your usual head/tail split in a functional language is [just `slice.split_at_mut(1)`][split]. For a long time, Rust had an experimental system for pattern matching on slices which was super cool, but the feature was simplified when it was stabilized. Still, [basic slice patterns][slice-pats] are neat! And of course, slices can be turned into iterators!

*하지만 불변의 의미로 제한되어 있다면 링크된 목록이 매우 유용할 수 있습니다*.

*But yes, if you're limited to immutable semantics, linked lists can be very nice*.

함수형 프로그래밍이 반드시 약하거나 나쁘다고 말하는 것은 아닙니다. 하지만 근본적으로 의미론적으로 제한이 있습니다. 사물이 어떻게 *있는*지에 대해서만 이야기할 수 있고, 어떻게 *해야 하는지에 대해서는 이야기할 수 없습니다. 컴파일러가 수많은 [이색적인 변환][ghc]을 할 수 있고 사용자가 걱정할 필요 없이 작업을 수행하는 *최선의* 방법을 알아낼 수 있기 때문에 이것은 사실 *기능*입니다. 그러나 이는 걱정할 수 있다는 대가가 따릅니다. 일반적으로 탈출구가 있지만 어느 정도 한계에 도달하면 다시 절차적 코드를 작성해야 합니다.

Note that I'm not saying that functional programming is necessarily weak or bad. However it *is* fundamentally semantically limited: you're largely only allowed to talk about how things *are*, and not how they should be *done*. This is actually a *feature*, because it enables the compiler to do tons of [exotic transformations][ghc] and potentially figure out the *best* way to do things without you having to worry about it. However this comes at the cost of being *able* to worry about it. There are usually escape hatches, but at some limit you're just writing procedural code again.

함수형 언어를 사용하더라도 실제로 데이터 구조가 필요할 때는 작업에 적합한 데이터 구조를 사용하도록 노력해야 합니다. 단일 연결 리스트는 제어 흐름을 위한 기본 도구이기는 하지만 실제로 많은 데이터를 저장하고 쿼리하는 데는 매우 열악한 방법입니다.

Even in functional languages, you should endeavour to use the appropriate data structure for the job when you actually need a data structure. Yes, singly-linked lists are your primary tool for control flow, but they're a really poor way to actually store a bunch of data and query it.


## 연결 리스트는 동시 데이터 구조를 구축하는 데 유용합니다! Linked lists are great for building concurrent data structures!

그렇습니다! 하지만 동시 데이터 구조를 작성하는 것은 완전히 다른 문제이며, 가볍게 생각해서는 안 됩니다. 확실히 많은 사람이 *고려*하지도 않을 것입니다. 일단 작성된 후에는 연결 리스트를 사용하도록 선택하는 것이 아닙니다. MPSC 큐를 사용하거나 다른 방법을 선택하게 됩니다. 이 경우 구현 전략은 꽤 멀리 떨어져 있습니다!

Yes! Although writing a concurrent data structure is really a whole different beast, and isn't something that should be taken lightly. Certainly not something many people will even *consider* doing. Once one's been written, you're also not really choosing to use a linked list. You're choosing to use an MPSC queue or whatever. The implementation strategy is pretty far removed in this case!

하지만 연결 리스트는 잠금 없는 동시성의 어두운 세계에서 사실상의 영웅입니다.* * 그렇습니다.

*But yes, linked lists are the defacto heroes of the dark world of lock-free concurrency.*




## 멈블멈블 커널에 무언가 방해가 되는 것이 포함되었습니다. Mumble mumble kernel embedded something something intrusive.

틈새 시장입니다. 언어의 *런타임*도 사용하지 않는 상황에 대해 이야기하고 있는 거군요. 뭔가 이상한 일을 하고 있다는 적신호가 아닐까요?

It's niche. You're talking about a situation where you're not even using your language's *runtime*. Is that not a red flag that you're doing something strange?

또한 매우 안전하지 않습니다.

It's also wildly unsafe.

*하지만 확실합니다. 스택에서 멋진 제로 할당 목록을 작성하세요.

*But sure. Build your awesome zero-allocation lists on the stack.*





## 이터레이터는 관련 없는 삽입/제거로 인해 무효화되지 않습니다. Iterators don't get invalidated by unrelated insertions/removals

정말 섬세한 춤이네요. 특히 가비지 컬렉터가 없다면 더욱 그렇습니다. 세부 사항에 따라서는 제어 흐름과 소유권 패턴이 너무 복잡하게 얽혀 있다고 주장할 수도 있습니다.

That's a delicate dance you're playing. Especially if you don't have a garbage collector. I might argue that your control flow and ownership patterns are probably a bit too tangled, depending on the details.

*하지만 커서를 사용하면 정말 멋지고 기발한 작업을 할 수 있습니다.

*But yes, you can do some really cool crazy stuff with cursors.*





## 간단하고 교육용으로도 훌륭합니다! They're simple and great for teaching!

네, 그렇죠. 그 전제에 관한 책을 읽고 계시잖아요. 단일 연결 리스트는 꽤 간단합니다. 하지만 이중으로 연결된 리스트는 좀 더 복잡해질 수 있습니다.

Well, yeah. You're reading a book dedicated to that premise. Well, singly-linked lists are pretty simple. Doubly-linked lists can get kinda gnarly, as we'll see.




# 숨 쉬기 Take a Breath

Ok. 이제 됐어요. 이제 수많은 연결 리스트를 작성해 봅시다.

[첫 번째 챕터로!](first.md)


[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/rust-unofficial/too-many-lists
[bjarne]: https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/std/primitive.slice.html
[split]: https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut
[slice-pats]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/
