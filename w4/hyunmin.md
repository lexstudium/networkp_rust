# Introducing reqwest

`reqwest` crate는 프로그래밍으로 통해 서버에 접속할 수 있게 해준다. 파이썬을 써본 사람이라면 RESTful 라이브러리인 `requests`와 굉장히 비슷하게 동작한다는 것을 알 수 있다.

## reqwest-example

일단 책에 나온 예제를 살펴보자.
```toml
# Cargo.toml

[package]
name = "reqwest-example"
version = "0.1.0"
edition = "2021"

[dependencies]
reqwest = { version = "0.11.9", features = ["json"] }
serde = "1.0.21"
serde_derive = "1.0.21"
serde_json = "1.0.6"
tokio = { version = "1.15.0", features = ["full"] }
futures = "0.3.19"

openssl-sys = "0.9"
openssl = "0.10"
```
책과는 많이 다른데 이유는 다음 파트에서 설명할 예정이다.

책에 나온 `reqwest`의 버전은 openssl과의 호환성 때문에 쓰지 못하니 꼭 둘 다 최신으로 사용해야 한다.

```rust
extern crate serde_json;
#[macro_use]
extern crate serde_derive;
extern crate reqwest;

#[derive(Debug,Serialize, Deserialize)]
struct Post {
    title: String,
    body: String,
    pinned: bool,
}

fn main() {
    let url = "http://localhost:8000/posts";
    let post: Post = Post {title: "Testing this".to_string(), body: "Try to write something".to_string(), pinned: true};
    let client = reqwest::Client::new();
    let res = client.post(url)
            .json(&post)
            .send()
            .unwrap();
    println!("Got back: {}", res.status());

    let mut posts = client.get(url).send().unwrap();
    let json: Vec<Post> = posts.json().unwrap();
    for post in json {
        println!("{:?}", post);
    }
}
```

이번 주에 다루지 않은 바로 이전 주제에서 만들어진 `Rocket` 서버와의 통신을 보여주고 있다. 일단 내부 구현을 모르고 있다는 가정을 하고 어떻게 구성되어 있는지를 살펴보자.

서버에 post를 만들고 서버로부터 post 데이터를 받아온 뒤 출력하기 위한 post struct가 있다. 그리고 `Serde` crate를 사용해 데이터를 JSON 포맷으로 serialization, deserialization 해서 주고 받을 것이다. `send()` 함수는 데이터를 지정된 url에 보내고 결과값을 리턴한다.

### simple self-made reqwest example

```rust
use serde_derive::{Serialize, Deserialize};
use reqwest;
use tokio;

#[derive(Debug, Serialize, Deserialize)]
struct Date {
    time: String,
    milliseconds_since_epoch: u64,
    date: String,
}

// tokio let's us use "async" on our main function.
// If a function didn't be declared by "async",
// we couldn't use the "await" keyword inside of it.
#[tokio::main]
async fn main() {
    let url = "http://date.jsontest.com/";
    let client = reqwest::Client::new();

    // Retrieves a JSON object with the current date and time in human-readable form,
    // and the current number of milliseconds since UNIX epoch.
    let res = client.get(url)
                    .send()
                    .await
                    .unwrap()
                    .json::<Date>()
                    .await
                    .unwrap();
    println!("{:?}", res);
    println!("{:?} {:?} {:?}", res.time, res.milliseconds_since_epoch, res.date);
}
```
간단하게 JSON을 리턴해주는 사이트가 있어서 이를 이용해 결과값만을 받아 출력하는 프로그램을 작성했다. 그런데 보면 알겠지만 책의 예제와는 많이 동떨어진 코드라는 것을 바로 알 수 있다. 구조가 많이 바뀐 모양인지 `async/.await`를 사용하는 `Future` trait이 요긴하게 쓰이는 모습이다.

## Asynchronous programming using tokio

다음으로는 `tokio`를 사용한 비동기 프로그래밍을 소개하는 파트다. 하지만 이미 위에서 보여주었다시피 기본적으로 비동기를 사용하도록 바뀌어서 크게 의미는 없는 파트다.

다시 한 번 위에서 보여주었던 내가 작성한 `Cargo.toml`을 보도록 하자.

```toml
# Cargo.toml

[package]
name = "reqwest-example"
version = "0.1.0"
edition = "2021"

[dependencies]
reqwest = { version = "0.11.9", features = ["json"] }
serde = "1.0.21"
serde_derive = "1.0.21"
serde_json = "1.0.6"
tokio = { version = "1.15.0", features = ["full"] }
futures = "0.3.19"

openssl-sys = "0.9"
openssl = "0.10"
```
책에서 사용한 `Cargo.toml`과 꽤나 달라진 모습이다. 진화된 모습이라고 보면 될 것 같다.

예전 버전의 `reqwest`에는 비동기 기능이 없어 여러 다른 crates의 기능을 끌어다가 개발자가 직접 비동기에 대한 메소드들을 구현해줘야했다. 하지만 documentation에서도 확인할 수 있듯이 정말 간단하게 비동기를 사용할 수 있도록 바뀌었으니 확인해보도록 하자.


# References

- [reqwest doc](https://docs.rs/reqwest/0.11.9/reqwest/)
- [tokio doc](https://docs.rs/tokio/1.15.0/tokio/)
- [Making HTTP requests in Rust with Reqwest](https://blog.logrocket.com/making-http-requests-rust-reqwest/)
- [Rust의 async/await와 Future](https://showx123.tistory.com/85)
