# Asynchronous Programming

Rust의 asynchronous programming은 기본적으로 `async`, `await`를 이용해서 `Future`를 주고받는 형식으로 진행됩니다. `Future<Output = T>` trait는 계산이 끝났을 수도 있고 안 끝났을 수도 있는 값을 갖는 타입입니다.

정리를 하다가 여기서부터 큰 난관에 부딪혔습니다. Rust 언어의 `Future` trait를 검색해보면 `std::future::Future`와 `futures::future::Future` 이렇게 2가지가 나옵니다. 책에서는 후자의 구버전을 사용하고 있습니다. 근데 std에 포함돼 있으면 왜 굳이 외부 크레이트를 쓸까하는 생각이 들어서 두 trait의 문서를 모두 읽어보았습니다. 아주 비슷하게 생겼더군요.

그래서 더 찾아보았습니다. [std vs library future](https://book.async.rs/overview/std-and-library-futures.html)
> 간단히 요약하자면, 원래 `futures::future::Future`는 외부 크레이트였습니다. 근데 `async`와 `await`라는 키워드가 Rust의 정식 문법에 들어오면서 `Future` trait가 필요해졌고, 그래서 외부 크레이트인 `futures`의 일부를 `std`에 편입시켰습니다. 또한, 일반 프로그래머 입장에서는 `std::future::Future`는 사용할 일이 없고, 모든 작업은 `futures` 크레이트를 통해서 하면 됩니다.

## Future

웬만해선 책에 있는 예제를 이용해서 `futures::future::Future`에 대해서 설명하고 싶었지만, 책이 옛날 책인지라 `async`, `await`를 전혀 사용하지 않았더군요. 그래서 요즘 트렌드에 맞게 수정을 좀 했습니다.

```rust
use futures::future::Future;

async fn func0() -> u8 { 5 }

fn func1() -> impl Future<Output = u8> { async {5} }

fn func2() -> impl Future<Output = u8> { futures::future::ready(5) }
```

책에서는 `func2`와 비슷한 방식으로 타입을 정하지만, 요즘은 `async`, `await` 키워드를 이용해서 훨씬 간편하게 할 수 있습니다. 참고로 위의 세 함수는 모두 컴파일 에러가 나지 않습니다.

`async`와 `await` 키워드를 이용해서 책의 예제와 최대한 비슷한 코드를 짜보겠습니다.

```rust
async fn very_stupid_async_function(r: u64) -> u64 {
    println!("here we go!");

    let mut x = 0;
    let mut y = r;
    let rs = r * r;
    let mut result = 0;

    while y > 0 {

        while x * x + y * y > rs {
            y -= 1;
        }

        result += y;
        x += 1;
    }

    result * 4
}
```

아주 단순무식한 함수를 만들었습니다. 함수 실행 순서를 보기 쉽도록 함수의 첫 줄에 print문을 하나 넣었습니다.

```rust
fn main() {

    let input = 1_000_000_000;

    println!("Right before first call");

    let res_one = very_stupid_async_function(input);
    println!("Called async_function");

    let res_two = very_stupid_async_function(input);
    println!("Called async_funcion");

    println!("Results are {} and {}", res_one, res_two);
}
```

이 상태 그대로 실행을 하면 컴파일이 되지 않습니다. `res_one`과 `res_two`의 타입이 `u64`가 아니고 `Future<Output = u64>`이기 때문입니다. 즉, 아직 계산은 시작하지도 않은 상태입니다.

```rust
println!("Results are {} and {}", res_one.await, res_two.await);
```

마지막 결과 부분에 `await` 키워드를 추가하여 보았지만 여전히 컴파일 에러가 뜹니다.

```
error[E0728]: `await` is only allowed inside `async` functions and blocks
```

라는 에러 메시지가 뜨는데, `main` 함수가 `async` 함수가 아니어서 그렇습니다. 그렇다면 `main` 함수 앞에 `async` 키워드를 붙이면 될까요??

```rust
async fn main() {
    // 생략...

    println!("Results are {} and {}", res_one.await, res_two.await);
}
```

를 컴파일 해보면

```
`main` function is not allowed to be `async`
```

라는 에러와 함께 죽습니다. `main`은 특별한 함수라서 양념 치는 걸 싫어하나 보군요. 이를 해결할 수 있는 방법은 여러가지가 있습니다. 가장 흔하게 사용하는 방법은 `tokio` 크레이트를 사용하는 것입니다. `tokio`는 asynchronous한 실행을 위한 런타임을 기본적으로 제공합니다. 이를 사용하면 그냥 `main`을 `async fn main()`처럼 사용할 수 있습니다.

혹은, `futures` 크레이트에서 제공하는 다양한 함수들을 사용할 수도 있습니다.

```rust
use futures::executor::block_on;

fn main() {
    block_on(async_main());
}

async fn async_main() {

    let input = 100_000;

    println!("Right before first call");

    let res_one = very_stupid_async_function(input);
    println!("Called async_function");

    let res_two = very_stupid_async_function(input);
    println!("Called async_funcion");

    println!("Results are {} and {}", res_one.await, res_two.await);
}
```

`futures::executor::block_on`을 호출하면 현재 스레드를 block하고 주어진 `async` 함수를 실행한 뒤 그 값을 반환합니다.

```
Right before first call
Called async_function
Called async_funcion
here we go!
here we go!
Results are 3141592653589764828 and 3141592653589764828
```

출력 결과물을 보면 `await`가 나올 때까지 실행되지 않음을 볼 수 있습니다. 다만 이런 식으로 실행해서는 asynchronous programming의 혜택을 전혀 보지 못합니다. `res_one`의 계산이 끝날 때까지 기다린 이후에 `res_two`를 계산하기 때문이죠. 좀 더 효율적으로 하려면 `futures::join!` 매크로를 사용할 수 있습니다.

```rust
println!("{:?}", futures::join!(res_one, res_two));
```

마지막 출력문만 다음과 같이 바꾸면 두 값을 동시에 계산한 뒤 tuple로 묶어서 반환합니다.
