# Socket multiplexing in tokio

서버가 비동기로 request를 처리하는 모델 중 하나는 `multiplexing`을 통해 처리하는 것이다.

- multiplexing이란?
	- 여러 단말기의 신호를 하나의 전송로에 중첩시켜 고속 신호 하나를 만들어 전송하는 방식
	- 전송로의 이용 효율이 높아짐
	- [ref](https://en.wikipedia.org/wiki/Multiplexing)

이 모델을 기반으로 한 서버는 다양한 복잡성을 가지는 다수의 incoming requests를 잘 처리해야하는 책임을 가진다. Unix에서는 `select`, `poll` system calls를 통해 socket multiplexing을 지원했다면, tokio ecosystem에서는 다수의 traits를 사용해서 multiplexed protocols를 구현한다.

### Cargo.toml

```toml
[package]
name = "collatz-multiplexed"
version = "0.1.0"
edition = "2021"

[dependencies]
bytes = "1.1.0"
slab = "0.4.5"
tokio = { version = "1.16.1", features = ["full"] }
tokio-util = { version = "0.6.9", features = ["codec"] }  # tokio_io::codec
tokio-tower = "0.6.0"  # tokio_proto::multiplex, tokio_service::Service
tower-service = "0.3.1"   # Service trait for Tower crate
futures-util = "0.3.19"   # common utilities and extensions traits for futures-rs library
serde = { version = "1.0", features = ["derive"] }
async-bincode = "0.6.1"   # alternative of tokio-util::codec (same as deprecated crate, tokio-codec)
```

## lib.rs

```rust
use serde::{Serialize, Deserialize};

use futures_util::future::poll_fn;
use std::task::{Context, Poll};
use tower_service::Service;

/* MultiplexTransport를 통해 초기화된 Client 인스턴스 내 poll_ready 호출.
 * 비동기 task를 나타내는 Context(cx)를 closure로 받아 루틴 처리.
 */
pub async fn ready<S: Service<Request>, Request>(svc: &mut S) -> Result<(), S::Error> {
  poll_fn(|cx| svc.poll_ready(cx)).await
}

/* [struct Request]
 * request를 보낼 때 collatz sequence의 시작값으로 value를 초기화.
 */
#[derive(Serialize, Deserialize)]
pub struct Request {
  tag: usize,
  value: u32,
}

impl Request {
  pub fn new(val: u32) -> Self {
    Request { tag: 0, value: val }
  }

  pub fn check(&self, expected: u32) {
    assert_eq!(self.value, expected);
  }
}

/* [struct Response]
 * 서버 측에서 보낼 response 포맷.
 */
#[derive(Debug, Serialize, Deserialize)]
pub struct Response {
  tag: usize,
  value: u32,
  collatz_str: String,
}

impl From<Request> for Response {
  fn from(r: Request) -> Response {
    Response {
      tag: r.tag,
      value: r.value,
      collatz_str: String::new(),
    }
  }
}

impl Response {
  pub fn check(&self, expected: u32) {
    assert_eq!(self.value, expected);
  }

  pub fn get_tag(&self) -> usize {
    self.tag
  }

  pub fn get_collatz_str(&self) -> &str {
    &self.collatz_str
  }

  #[warn(deprecated)]
  fn set_collatz_str(&mut self, s: &str) {
    self.collatz_str = s.to_string();
  }

  fn new(r: Request, v: Vec<u32>) -> Response {
    Response {
      tag: r.tag,
      value: r.value,
      collatz_str: format!("{:?}", v),
    }
  }
}

impl Request {
  pub fn set_tag(&mut self, tag: usize) {
    self.tag = tag;
  }
}

/* [unit-like struct PanicError]
 * tokio_tower::multiplex::client::Client 용 Error handler
 */
pub struct PanicError;
use std::fmt;
impl<E> From<E> for PanicError
where
    E: fmt::Debug,
{
  fn from(e: E) -> Self {
    panic!("{:?}", e)
  }
}

/* [fn unwrap]
 * user-defined unwrap function 
 * that is modified from original unwrap function
 */
pub fn unwrap<T>(r: Result<T, PanicError>) -> T {
  if let Ok(t) = r {
    t
  } else {
    unreachable!();
  }
}

/* [fn get_sequence]
 * Collatz Sequence generator
 */
fn get_sequence(mut n: u32) -> Vec<u32> {
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

/* [unit-like struct CollatzService]
 * 서버에서 사용할 Service 정의
 */
pub struct CollatzService;

impl Service<Request> for CollatzService {
  type Response = Response;
  type Error = ();
  type Future = futures_util::future::Ready<Result<Self::Response, Self::Error>>;

  fn poll_ready(&mut self, _: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    Poll::Ready(Ok(()))
  }

  fn call(&mut self, req: Request) -> Self::Future {
    let seq = get_sequence(req.value);
    futures_util::future::ok(Response::new(req, seq))
  }
}
```

## server.rs

incoming requests 각각에 unique ID를 부여해서 되돌려주는 responses가 헷갈리지 않도록 만든다. client에서 server로 보내는 데이터 전송 유닛의 형태는 `[RequestID (4B) | data | \n (1B)]`와 같다... 라고 책에서는 독자적인 코덱 프레임을 정해서 Encoder, Decoder를 구현했다. 하지만 `tokio-tower` crate(기존 tokio 내에서 구현됐지만 너무 복잡하고 무거워져 내부에서 빠진 후 차기 multiplex를 호환할 수 있는 crate를 토론을 통해 결정한게 tower crate였고 그 호환물이 바로 tokio-tower crate다)의 모듈 중 하나로 multiplex가 있고 지원하는 기능 중 하나가 Tag를 붙이는 것이다. 이는 RequestID를 붙이는 것과 같은데 결국 우리는 책에서 소개한대로 개발하지 않아도 된다. 덧붙여서 그렇기 때문에 해당 데이터들은 codec을 구현하기 위한 crate(tokio에서 떨어져서 생긴 unversioned features가 모여 있는 tokio-util 같은 것)를 따로 사용하지 않고 doc에서도 추천한 crate 중 하나인 `AsyncBincode` crate를 이용한다.

```rust
use collatz_multiplexed::CollatzService;
use async_bincode::*;
use tokio::net::{TcpListener};
use tokio_tower::multiplex::Server;

#[tokio::main]
async fn main() {
  let rx = TcpListener::bind("0.0.0.0:8888").await.unwrap();
  
  // accept
  let (rx, _) = rx.accept().await.unwrap();
  let rx = AsyncBincodeStream::from(rx).for_async();
  let server = Server::new(rx, CollatzService);

  tokio::spawn(async move { server.await.unwrap() });
}
```

## client.rs

```rust
use collatz_multiplexed::{ready, unwrap, PanicError, Request, Response};
use slab::Slab;
use async_bincode::*;
use tokio::net::TcpStream;
use tokio_tower::multiplex::{Client, MultiplexTransport, TagStore};
use tower_service::Service;

use std::pin::Pin;

/* [pub(crate) struct SlabStore]
 * for dealing with tag using in multiplex.
 */
pub(crate) struct SlabStore(Slab<()>);

impl TagStore<Request, Response> for SlabStore {
  type Tag = usize;

  fn assign_tag(mut self: Pin<&mut Self>, request: &mut Request) -> usize {
    let tag = self.0.insert(());
    request.set_tag(tag);
    tag
  }

  fn finish_tag(mut self: Pin<&mut Self>, response: &Response) -> usize {
    let tag = response.get_tag();
    self.0.remove(tag);
    tag
  }
}

#[tokio::main]
async fn main() {
  // connect
  let tx = TcpStream::connect("127.0.0.1:8888").await.unwrap();
  let tx = AsyncBincodeStream::from(tx).for_async();
  let mut tx: Client<_, PanicError, _> = 
    Client::new(MultiplexTransport::new(tx, SlabStore(Slab::new())));

  unwrap(ready(&mut tx).await);
  let fut1 = tx.call(Request::new(110));
  println!("{}", unwrap(fut1.await).get_collatz_str());
}
```

---

## References

 - https://github.com/tower-rs/tokio-tower/blob/master/tests/lib.rs
 - https://github.com/tower-rs/tokio-tower/blob/master/tests/multiplex/mod.rs
