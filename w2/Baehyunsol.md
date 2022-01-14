# Concurrency

멀티 스레딩은 어렵습니다. 메모리를 언제 할당/해제할지, data-race는 어떻게 막을지 생각할 게 아주 많습니다. 다행스럽게도 Rust의 강력한 컴파일러의 도움을 받으면 멀티스레딩을 쉽게 다룰 수 있습니다.

Rust는 low-level한 언어를 지향하기 때문에 OS와 1:1 대응되는 스레딩만 지원합니다. Go 언어처럼 OS 스레드 하나에 많은 고루틴을 집어 넣는 방식은 없다네요.

## std::thread

```rust
use std::thread;

fn main() {

    for i in 1..10 {
        let handle = thread::spawn(move || {
            println!("Hello from thread number {}", i);
        });

        let _ = handle.join();
    }

}
```

아주 간단한 예시입니다. `thread::spawn`은 클로져 하나를 인수로 받아서, 새로운 스레드에서 해당 클로져를 실행시킵니다. `thread::spawn`은 `JoinHandle`이라는 type을 return합니다.
그 다음 줄의 `handle.join()`은 방금 만든 스레드의 실행이 끝날 때까지 다른 실행들을 전부 멈춥니다.

위의 프로그램을 실행하면 아래와 같은 결과가 나옵니다.

```
Hello from thread number 1
Hello from thread number 2
Hello from thread number 3
Hello from thread number 4
Hello from thread number 5
Hello from thread number 6
Hello from thread number 7
Hello from thread number 8
Hello from thread number 9
```

만약 `handle.join()`이 없다면 어떻게 될까요?

```rust
for i in 0..30 {
    thread::spawn(move || {println!("Hello from thread number {}", i);});
}
```

원래 code의 `for`문을 위와 같이 바꿔서 실행해보겠습니다.

```
Hello from thread number 0
Hello from thread number 1
Hello from thread number 2
Hello from thread number 4
Hello from thread number 6
Hello from thread number 3
Hello from thread number 5
Hello from thread number 7
Hello from thread number 9
Hello from thread number 10
```

`join` 메소드가 없으면 실행할 때마다 다른 결과가 나오는데, 그 중에서 가장 설명하기 좋은 결과를 들고 왔습니다.
이상한 결과가 나왔죠? 스레드가 어떤 순서로, 어떤 타이밍에 실행될지는 OS가 결정하기 때문에 만들어진 순서와 다르게 실행될 때도 있습니다. 또한, 11번째 스레드가 실행되기 전에 메인 스레드의 `main` 함수가 끝나버려서 다른 모든 스레드들도 실행되지 않고 죽어버렸네요.

## Channels

지금까지 봤던 내용으로는 복잡한 프로그램을 짜기 힘들어보입니다. 스레드를 여러 개 띄워 놓고 걔네들끼리 정보를 주고 받으면서 일을 처리하게 하려면 무슨 방법을 써야할까요? Rust의 standard library는 `std::sync::mpsc`라는 모듈을 지원합니다. 간단히 설명하자면, `mpsc`는 여러 스레드들끼리 메시지를 주고 받을 수 있는 채널입니다.

```rust
use std::thread;
use std::sync::mpsc;

fn main() {

    let (tx, rx) = mpsc::channel();

    for i in 1..10 {
        let tx = tx.clone();

        let handle = thread::spawn(move || {
            let s = format!("{} + {} = {}", i, i, i + i);
            tx.send(s).unwrap();
        });

        let _ = handle.join();
    }

    drop(tx);

    for result in rx {
        println!("{}", result);
    }
}
```

`tx`와 `rx`를 제외한 부분은 이전에 봤던 코드와 비슷합니다.

`let (tx, rx) = mpsc::channel();`은 `mpsc::channel();`을 이용해서 메시지를 주고 받을 수 있는 channel을 만든 뒤 송신자와 수신자를 각각 `tx`, `rx`에 할당합니다.

`for`문 안에서 매번 `tx.clone()`을 호출합니다. 새로 만든 스레드에서 `tx`를 사용하기 위해선 `tx`의 소유권을 해당 스레드에 통째로 넘겨야하는데, clone 없이 넘겨버리면 `for`문에서 더이상 해당 `tx`를 사용할 수 없기 때문이죠.

스레드 안에서 `tx.send(s)`를 호출합니다. 그러면 그 스레드의 결과값인 `s`를 채널을 통해서 다른 스레드로 넘길 수 있습니다. 각 스레드들이 넘긴 값들은 마지막에 메인 스레드에서 `for result in rx`를 이용해서 전부 확인합니다.

`rx`의 값을 확인하기 전에 `drop(tx)`를 이용해서 `tx`를 해제합니다. `drop` 함수는 C/C++의 메모리 해제함수와 비슷한 역할을 합니다. `tx`가 살아있는 동안은 계속 메시지 송/수신을 대기하기 때문에 프로그램이 종료되지 않습니다. 실제로 `drop` 함수를 제거하고 코드를 실행시켜보면 결과는 정상적으로 출력되지만 프로그램이 종료되지 않는 것을 확인할 수 있습니다.

위 코드의 실행 결과는 아래와 같습니다.

```
1 + 1 = 2
2 + 2 = 4
3 + 3 = 6
4 + 4 = 8
5 + 5 = 10
6 + 6 = 12
7 + 7 = 14
8 + 8 = 16
9 + 9 = 18
```

## Send, Sync

channel을 통해서 아무 값이나 주고받을 수 있을까요? C/C++에서는 멀티스레딩 하면서 아무 값이나 주고받다가 탈나는 경우가 많은데 Rust도 그럴까요? Rust 컴파일러가 소유권 검사, 타입 검사를 빡세게 해주기 때문에 걱정하지 않아도 됩니다.

`let a = vec![1, 2, 3, 4]`라는 값을 여러 스레드에서 동시에 사용한다고 생각해봅시다. Rust의 소유권 검사 덕분에 `a`의 값을 수정할 수 있는 대상은 하나밖에 없습니다. 즉, `a`의 소유권이나 `&mut`을 가진 함수는 하나밖에 없기 때문에 `a`의 값에 data-race가 일어날 걱정을 하지 않아도 됩니다.

그렇다면 소유권 검사를 피해가는 타입들은 어떨까요? Rust에는 `Rc`라는 포인터가 있습니다. `Rc`는 reference-count 포인터로, 해당 포인터가 clone되면 reference-count를 1 늘리고, drop되면 reference-count를 1 감소시킵니다. 즉, `Rc`의 소유권이 없는 함수들도 `Rc`의 reference-count를 감소시키기 위해서 `Rc`의 값을 수정할 수 있습니다. 그럼 여러 스레드에서 한 `Rc`의 reference-count를 동시에 수정하려고 하면 어떻게 될까요? 안되겠죠? 그래서 그냥 `Rc`는 멀티 스레딩에서 사용할 수 없습니다.

`Rc`처럼 싱글스레드에선 안전하지만 멀티스레드에선 위험한 타입들을 관리하기 위해서 Rust에는 `Send`, `Sync`라는 trait들이 있습니다. 어떤 값을 송신하려면 그 값은 `Send` trait가 구현돼 있어야 하고, 어떤 값을 수신하려면 `Sync` trait가 구현돼 있어야 합니다. 직접 구현할 필요는 없습니다. Rust의 기본 type들의 `Send`, `Sync`는 컴파일러가 관리해줍니다. 새로 만든 type의 `Send`, `Sync`도 컴파일러의 타입검사장치가 알아서 확인해주니 걱정할 필요 없습니다. 단지, `Send`가 없는 타입을 이용해서 새로운 타입을 만들었을 경우 그 타입도 `Send`가 없다는 것만 유의하시면 됩니다.

아래의 코드로 예시를 들겠습니다.

```rust
use std::rc::Rc;

struct Point {
    x: f32,
    y: f32
}

struct NoSend {
    x: Rc<f32>,
    y: Rc<f32>
}
```

`f32` 타입은 `Send`, `Sync`가 전부 구현돼 있어 스레드 간에 주고받는 게 가능하지만 `Rc`는 그렇지 않습니다. 그래서 `f32`로만 이뤄진 `Point`는 `Send`와 `Sync`가 자동으로 구현되지만 `NoSend` 타입은 그렇지 않습니다.

# unsafe Rust

Rust 언어의 빡빡한 규칙은 양날의 검입니다. 런타임에 일어날 버그들을 미리 잡아주는 효과도 있지만 프로그래머의 자유를 떨어뜨리기도 합니다. 가끔은, Rust의 규칙 안에서는 구현이 절대 불가능한 코드들도 있습니다(아주 low-level인 경우). 또 가끔은, 소유권 검사 어쩌구 저쩌구 하기 귀찮고 C/C++처럼 포인터 남발하고 싶을 때도 있습니다.

그래서 Rust에는 `unsafe`라는 키워드가 존재합니다. `unsafe`를 통해서 Rust 언어의 빡빡한 규칙들을 완화할 수 있습니다. 다만, `unsafe`를 사용한다고 해서 컴파일러가 아무 검사도 하지 않는 것은 아닙니다. `unsafe` 블록 안에서도 기본적인 소유권 검사, 타입 검사는 전부 합니다. `unsafe` 블록 안에서 새롭게 할 수 있는 행동은 4 종류가 있습니다.

- dereference raw pointer
  - C/C++에서 쓰던 단순무식한 포인터
- unsafe한 함수/메소드 호출
- unsafe한 trait 구현
- mutate static variables
  - C/C++의 전역 변수와 거의 비슷한 느낌입니다.

```rust
fn main() {

    let num: u32 = 42;
    let p: *const u32 = &num;

    unsafe {
        assert_eq!(*p, num);
    }

}
```

`p`의 타입은 `*const u32`, 즉 `u32`에 대한 raw pointer입니다. `*p`로 `p`의 값을 확인하는 작업은 `unsafe` 블록 안에서만 할 수 있습니다.

# Testing

Rust 언어는 테스트도 아주 잘 구현돼 있습니다. 아래의 코드와 함께 설명하겠습니다.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test1() {
        assert_eq!(1 + 1, 2);
    }

    #[test]
    fn test2() {
        assert_eq!(1 + 1, 3);
    }

    #[test]
    fn test3() {
        panic!("test3 fails!");
    }
}
```

새로운 library를 만들면 `tests`라는 모듈이 자동으로 생성되고 모듈 위에는 `#[cfg(test)]`라는 매크로가 붙어 있습니다. `#[cfg(test)]`는 `cargo test`를 하면 해당 모듈을 실행하라는 뜻의 매크로입니다. `cargo test`를 실행하면 `tests` 모듈 안의 `#[test]`가 붙은 모든 함수들이 실행됩니다. 각각의 함수들은 서로 다른 스레드에서 실행 되는데, 어떤 함수에서 panic이 일어나게 되면 그 스레드가 통째로 죽어버립니다. 메인 스레드에서는 정상적으로 종료된 스레드와 그렇지 않은 스레드를 확인하여 테스트 중 몇개가 성공했는지 확인합니다. 위의 테스트를 실행한 결과는 아래와 같습니다.

```
running 3 tests
test tests::test1 ... ok
test tests::test2 ... FAILED
test tests::test3 ... FAILED

failures:

---- tests::test2 stdout ----
thread 'tests::test2' panicked at 'assertion failed: `(left == right)`
  left: `2`,
 right: `3`', src\lib.rs:10:9

---- tests::test3 stdout ----
thread 'tests::test3' panicked at 'test3 fails!', src\lib.rs:15:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::test2
    tests::test3

test result: FAILED. 1 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`test2`와 `test3`는 고의로 `panic`을 일으키도록 구현해놨고, 테스트 결과 화면에서 그걸 확인할 수 있습니다.
