# Heading to tokio

`Network Programming with Rust` 책에서 `tokio`에 관한 내용이 이제 등장하기 시작한다. 다만 책이 나온지 4년이상의 시간이 지났기 때문에 `tokio`도, 사용하는 `crate`들도 많이 바뀌었다.

[tokio-core, Depreceted](https://github.com/tokio-rs/tokio-core)  
[tokio-proto, Deprecated](https://github.com/tokio-rs/tokio-proto)

tokio-proto의 내용을 보자면 Tokio는 초기에 API기반의 고차원 request, response를 제공하고 I/O 관련은 구현 디테일 부분이였는데, 오늘날에는 Tokio의 관심사가 non-blocking I/O library로 바뀌었고 request/response관련은 [Tower](https://github.com/tower-rs/tokio-tower) 이라는 곳으로 이동했다.

우선 책에 나온 코드를 지금의 tokio버전에서 알아보자.

```rust
// Cargo.toml
[package]
name = "futures-loop"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use std::future::Future;
use std::io;
use std::io::BufRead;
use tokio::net::TcpListener;

async fn check_prime_boxed(n: u64) -> Box<dyn Future<Output = bool> + Unpin> {
    for i in 2..n {
        if n % i == 0 {
            return Box::new(std::future::ready(true));
        }
    }
    Box::new(std::future::ready(false))
}

#[tokio::main]
async fn main() {
    let stdin = io::stdin();
    loop {
        let mut line = String::new();
        stdin
            .lock()
            .read_line(&mut line)
            .expect("Could not read from stdin");
        let input = line
            .trim()
            .parse::<u64>()
            .expect("Could not parse input as u64");
        let result = check_prime_boxed(input).await;
        println!("Result: {:?}", result.await);
    }
}
```

그전과 Future Trait의 연관타입이 바뀌었다. 그전에는 Future Trait에 Item과 Error이라는 연관타입이 있었는데, 그것을 합쳐 Type Output으로 바뀌었다.
여기서 `check_prime_boxed(n)`의 리턴타입에 `+Unpin`이 붙어있는것을 확인할 수 있다. 그리고 이것이 없으면 컴파일 에러가 난다.

#### Unpin 제거시

`dyn Future<Output = bool>` cannot be unpinned
consider using `Box::pin`
required because of the requirements on the impl of `Future` for `Box<dyn Future<Output = bool>>`

`Box::new()` 대신에 `Box::pin()`을 사용하기를 권장한다. 또는 위의 코드처럼 Box에 `+Unpin`을 추가해줘야 한다. 그러면 `Unpin`이라는 것은 무엇일까?  
우선 `Pin`부터 알아보자.  
예시코드를 하나 보자.

```rust
// reference: https://rust-lang.github.io/async-book/04_pinning/01_chapter.html
#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn b(&self) -> &String {
        assert!(
            !self.b.is_null(),
            "Test::b called without Test::init being called first"
        );
        unsafe { &*(self.b) }
    }
}

fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    // a: test1, b: test1
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();

    println!("a: {}, b: {}", test2.a(), test2.b());
    // a: test1, b: I've totally changed now!
}
```

`Test`라는 구조체 에서 a와 b라는 필드가 있는데, b는 a의 주소를 담고 있습니다.  
여기서 `Test test1`과 `Test test2`를 선언하고, 두 변수를 swap했는데, swap후에도 여전히 b는 test1의 a의 주소를 가지고 있음을 출력 결과를 통해 확인할 수 있습니다.
![image](https://user-images.githubusercontent.com/71541643/152515049-83385b5e-e90c-4e18-93fb-f11fd5ab85c4.png)
이처럼 어떠한 객체가 복사되거나 대입되는 경우에 일반 값은 그냥 복사가 되면 되지만, 주소값을 따라가서 보면 엉뚱한 값이 들어가 있을 수 있다. 이러한 예상치 못한 행위를 방지하기위해 `Pin`이라는 트레잇이 나와 위처럼 내부포인터가 가리키는 값을 이동할 수 없도록 만듭니다. 그래서 이동에 안전한 타입에만 구현이 되어있습니다. 그와 반대로 `Unpin`은 위의 예시처럼 이동을 허용하는 트레잇 입니다. 대부분의 표준라이브러리 타입, 원시타입이 이를 구현하고 있습니다.
그리고 위와 같은 경우 Unpin이 자동으로 구현되어 구현되지 않도록 조치를 취하는게 필요한데, 이와 관련된 자세한 내용은 [https://neurowhai.tistory.com/371] 여기서 필요하신 경우 추가로 보시면 좋을것 같습니다.

그리고 비동기 프로그래밍에서 `await`은 `unpinned Future`을 기다리고 있는데, `Box <dyn Future<Output = bool>>` 만 적게되면 Box trait이 Unpin 트레잇을 구현하고 있음에도 불구하고 `unwrap()`을 하는 경우
![image](https://user-images.githubusercontent.com/71541643/152516039-46ad4778-6806-403e-b482-068227eceb82.png)

?Sized속성만 가지게 된다. 그래서 await이 예상하는 unpinned Future을 위해 `Box <dyn Future<Output = bool> + Unpin>`을 적어 `Future + Unpin`이 deref되어 이제 await을 할 수 있게 된다.

---

예전에는 tokio-proto에서 server와 client를 만들기위한 툴킷을 제공했었다.

1. codec 전송계층의 프로토콜 상에서 데이터가 어떻게 읽고 쓰여져야 하는지에 관한 약속.
1. protocol 코덱과 실제 event loop 사이에 있는 프로토콜로 simple reqeust-response type, multiplexed, streaming protocol등이 있다.
1. service 실제로 실행되는 함수 같은 서비스를 가지고 있는 층으로 대부분의 계산이 이루어진다.

이 세가지 layer을 tokio-proto를 사용하려면 이 세가지 layer을 구현하여야 했다. 하지만 tokio-proto는 이미 deprecated 되었다!

`Tokio`의 I/O는 `std`와 유사하게 사용할 수 있지만, 비동기적으로 동작한다. 읽기를 위한 `AsyncRead Trait`과 쓰기를 위한 `AsyncWrite Trait`이 있고, TcpStream, File, Stdout같은 특별한 타입이나, Vec<u8>이나 &[u8]과 같은 특별한 데이터구조에 구현되어있다. 이것들은 `Future trait`이 `poll`을 직접적으로 호출하지 않는것과 마찬가지로 각각의 trait에서 `poll_{read or write}` 를 호출하지 않고, 따로 제공하는 utility function을 이용한다. utility function으로는 `read(), read_to_end(), write(), write_all()`등이 있다.

```rust
// server.rs
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

fn get_sequence(n: u8) -> Vec<u8> {
    let mut n = n.clone();
    let mut result = vec![];
    result.push(n);
    while n > 1 {
        if n % 2 == 0 {
            n /= 2;
        } else {
            n = 3 * n + 1;
        }
        result.push(n);
    }
    result
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:3214").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let (mut rd, mut wr) = socket.split();
            let num = rd.read_u8().await.unwrap();
            println!("num: {}", num);
            let collatz_vec = get_sequence(num);
            wr.write_all(&collatz_vec[..]).await.unwrap();
        });
    }
}


// client.rs

use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:3214").await?;
    let (mut rd, mut wr) = io::split(socket);

    let send_number = 13;
    println!("Send: {}", send_number);
    tokio::spawn(async move {
        wr.write_u8(13).await?;

        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 1024];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
```

![image](https://user-images.githubusercontent.com/71541643/152501240-624d660e-c51e-4d7c-86fc-6a3b514827d7.png)

---

## Reference

1. https://stackoverflow.com/questions/60561573/how-can-one-await-a-result-of-a-boxed-future
1. https://neurowhai.tistory.com/371
1. https://rust-lang.github.io/async-book/04_pinning/01_chapter.html
1. https://tokio.rs/
