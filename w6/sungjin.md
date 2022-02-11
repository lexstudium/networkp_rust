# coroutines and generators

이번 장에서는 코루틴과 제너레이터에 관한 내용을 소개합니다.

코루틴(coroutine)은 cooperative routine를 의미하는데 서로 협력하는 루틴이라는 뜻입니다. 이 말은 메인 루틴과 서브 루틴처럼 종속된 관계가 아니라 서로 대등한 관계이며 특정 시점에 상대방의 코드를 실행하고 서로 값을 주고 받을 수도 있습니다.  
제너레이터는 세미코루틴이라고 불리는 코루틴의 일종으로, 코루틴과 마찬가지로 엔트리포인트가 여러곳 존재합니다.  
제너레이터는 값을 yield로 발생시키지만, 코루틴은 다른 루틴의 데이터를 전달받아 사용 가능합니다.  
책에서 추가적으로 2가지 타입의 코루틴을 소개하는데, 하나는 stackless coroutines 이고 나머지 하나는 stackful coroutines입니다.  
stackless는 이름 그대로 어디서 멈췄는지에 관한 정보를 스택에 저장하지 않는 반면에 stackful은 루틴의 정지 당시의 상황에 관한 정보를 stack에 보존합니다. 이 내용을 봤을때 쓰레드와 비슷하게 동작하는 느낌이지만 쓰레드와 프로세스보다 비용이 낮다고 합니다.

---

```rust
in std::ops;

pub trait Generator<R = ()> {
    type Yield;
    type Return;
    fn resume(
        self: Pin<&mut Self>,
        arg: R
    ) -> GeneratorState<Self::Yield, Self::Return>;
}

pub enum GeneratorState<Y, R> {
    Yielded(Y),
    Complete(R),
}
```

다음과 같이 제너레이터 Yield와 Return의 2가지 타입을 가지고 있습니다. 그리고 파이썬은 제너레이터를 호출할때 `.next()`를 사용하는데, 러스트는 `.resume()`을 통하여 마지막 flow로부터 이어서 실행을 하게 된다. 그리고 타입을 보면 알 수 있도록 Yield상태 일때 GeneratorState는 Yielded를, Return 상태일때 Complete로 나타냅니다.
다. 실제 사용되는 코드와 함께 제너레이터의 장점을 알아봅시다.

```rust
#![feature(generators, generator_trait)]

use std::{
    env,
    ops::{Generator, GeneratorState},
    pin::Pin,
};

fn main() {
    let input = env::args().nth(1).unwrap().parse::<u64>().unwrap();

    let mut gen = || {
        let mut current = input;
        let end = 100000000u64;
        while current != end {
            yield current;
            current += 1;
        }
        return end;
    };

    loop {
        match Pin::new(&mut gen).resume(()) {
            GeneratorState::Yielded(x) => {
                println!("{}", x);
            }
            GeneratorState::Complete(x) => {
                println!("{}", x);
                break;
            }
        }
    }
}
```

이런식으로 아주 큰 값을 가지는 것을 iterator로 가져온다고 생각해본다면, iterator로 쓰일 크기만큼의 배열을 선언하고 iterate할 수 있을것입니다. 그와 다르게 제너레이터는 값을 그때 그때 만들어 내기 때문에 메모리를 아낄 수 있다는 장점을 가집니다. 파이썬에서 머신러닝을 할 때 제너레이터를 통해 값을 전달 받기도 합니다.  
아쉽게도 지금은 nightly 버전에서만 제너레이터를 사용할 수 있습니다.

---

현재 러스트에서 많은 종류의 코루틴 구현체들이 있습니다. 그중 책에 소개된 것은 stackful coroutines중 `May`가 있습니다. `Go` 언어에 관하여 공부해본적이 없어서 잘 모르지만 그와 유사한 특성이 있고, macro를 통하여 동작한다고 합니다. Documentation에 Rust version of the popular Goroutine이라고 적혀있네요.

```rust
use generator::{done, Gn};
use may::go;
use std::env;

fn collatz_generator(start: u64) -> impl Iterator<Item = u64> {
    Gn::new_scoped(move |mut s| {
        let end = 1u64;
        let mut current: u64 = start;
        while current != end {
            s.yield_(current);
            if current % 2 == 0 {
                current /= 2;
            } else {
                current = 3 * current + 1;
            }
        }
        s.yield_(end);
        done!()
    })
}

fn collatz(start: u64) -> Vec<u64> {
    let end = 1u64;
    let mut current: u64 = start;
    let mut result = Vec::new();
    while current != end {
        result.push(current);
        if current % 2 == 0 {
            current /= 2;
        } else {
            current = 3 * current + 1;
        }
    }
    result.push(end);
    result
}

fn main() {
    let input = env::args()
        .nth(1)
        .expect("Please provide only one argument")
        .parse::<u64>()
        .expect("Could not convert input to integer");
    go!(move || {
        println!("{:?}", collatz(input));
    })
    .join()
    .unwrap();

    let results = collatz_generator(input);
    for result in results {
        println!("{}", result);
    }
}
```

우선 첫번째 함수가 impl Iterator을 리턴하고 있습니다. 이것은 제네릭 타입을 리턴하는 경우와 다르게, callee가 리턴타입을 정할 수 있습니다.

`may` 에서 위에 적은것과 같이 macro를 통해서 동작하게 되는데, `go!()`는 coroutine을 만드는 macro입니다. 이 덕분에 `::spawn(| | {})`과 같은 코드를 사용하지 않고 편리하게 `go!()`로 사용할 수 있습니다 (wrapper for spawn).

```rust
use may::coroutine;

let handler = unsafe {
    coroutine::spawn(|| {
        // coroutine code
    })
};

handler.join().unwrap();
```


---

그리고 마지막부분으로 `may`는 코루틴 바탕의 HTTP Library를 제공합니다. `hyper`의 구현 로직을 가져와서 인체공학적이고 퍼포먼스 개선을 위해 재작성했다고 합니다. 
`may-http`의 샘플 코드입니다.

```rust
// server
use http::header::*;
use may_http::server::*;

fn hello(_req: Request, rsp: &mut Response) {
    rsp.headers_mut()
        .append(CONTENT_TYPE, "text/plain; charset=utf-8".parse().unwrap());
    rsp.send(b"Hello World!").unwrap();
}

fn main() {
    let server = HttpServer::new(hello).start("127.0.0.1:8080").unwrap();
    server.wait();
}


// client
use http::Uri;
use std::io::{self, Read};
use may_http::client::*;

fn client_get(uri: Uri) -> io::Result<()> {
    let mut client = {
        let host = uri.host().unwrap_or("127.0.0.1");
        let port = uri.port().unwrap_or(80);
        HttpClient::connect((host, port))?
    };

    let mut s = String::new();
    for _ in 0..100 {
        let uri = uri.clone();
        let mut rsp = client.get(uri)?;
        rsp.read_to_string(&mut s)?;
        println!("get rsp={}", s);
        s.clear();
    }
    Ok(())
}

fn main() {
    let uri: Uri = "http://127.0.0.1:8080/".parse().unwrap();
    client_get(uri).unwrap();
}

```

---

Reference

1. https://www.ncameron.org/blog/dyn-trait-and-impl-trait-in-rust/
1. https://github.com/rust-may/may_http

