# Hyper

[Hyper](https://github.com/hyperium/hyper) is an HTTP framework written in Rust.

![benchmark](benchmark.jpg)
![benchmark2](benchmark2.jpg)

그림에서 보시면 243개의 웹 프레임워크 중 최상위권에 있는 것을 보실 수 있습니다. 야무지네요. 상위 20개의 프레임워크 중 Rust로 쓰인 프레임워크가 6개나 되는 것도 확인하실 수 있습니다.

![old](책버전.jpg)
![new](curr버전.jpg)

다만 아쉽게도 책에서 사용한 버전인 0.11.7이 너무 예전 버전이라 책 내용보다는 공식 문서의 내용을 위주로 서술하였습니다.

## Server

Hyper에는 client 쪽을 구현하는 부분과 server 쪽을 구현하는 부분이 각각 있습니다. server 쪽부터 보겠습니다.

```rust
use std::convert::Infallible;
use std::net::SocketAddr;
use hyper::{Body, Request, Response, Server};
use hyper::service::{make_service_fn, service_fn};
use hyper::server::conn::AddrStream;

#[derive(Clone)]
struct AppContext {
    // Whatever data your application needs can go here
}

async fn handle(
    context: AppContext,
    addr: SocketAddr,
    req: Request<Body>
) -> Result<Response<Body>, Infallible> {

    // context에는 현재 서버의 context가, 
    // addr에는 요청을 보낸 client의 주소가,
    // req에는 request가 들어 있습니다.
    // 그 정보들을 가지고 `Response<Body>`를 만들면 됩니다.
    Ok(Response::new(Body::from("Hello World")))
}

#[tokio::main]
async fn main() {
    let context = AppContext {
        // ...
    };

    // A `MakeService` that produces a `Service` to handle each connection.
    let make_service = make_service_fn(move |conn: &AddrStream| {
        // 멀티 쓰레드를 제대로 활용하기 위해선 `context`가 여러 쓰레드를 넘나들어야합니다.
        // 그래서 `clone` method를 호출했습니다. 혹시 `clone`이 구현되지 않았으면,
        // 지난주에 얘기했던 `std::sync::Arc`를 사용하시면 됩니다.
        let context = context.clone();

        // You can grab the address of the incoming connection like so.
        let addr = conn.remote_addr();

        // Create a `Service` for responding to the request.
        // client가 요청을 보낼 때마다 service_fn이 처리합니다.
        let service = service_fn(move |req| {
            handle(context.clone(), addr, req)
        });

        // Return the service to hyper.
        async move { Ok::<_, Infallible>(service) }
    });

    // Run the server like above...
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));

    let server = Server::bind(&addr).serve(make_service);

    if let Err(e) = server.await {
        eprintln!("server error: {}", e);
    }
}
```

위의 예시코드에 사용된 struct와 몇몇 함수들에 대해서 조금 더 깊이 알아보겠습니다.

### hyper::Request<T>

http request를 단순무식하게 문자열 그 자체로 다루지 않고, 추상화 레이어를 제공합니다.

type `T`에는 request의 body의 type이 들어갑니다. 문서에서는 `Vec<u8>`이나 `Stream`을 사용하라고 돼 있습니다. 다만, `hyper::Request`가 [serde](https://serde.rs/)와 호환이 되니 `T`도 `serde`와 호환이 되는 타입으로 고르시면 될 것 같습니다.

```rust
// 빈 body
let request: Request<()> = Request::builder()
    .method("GET")
    .uri("https://www.rust-lang.org/")
    .header("X-Custom-Foo", "Bar")
    .body(())
    .unwrap();

let request = Request::new("hello world");
```

### hyper::Body

아까 `hyper::Request<T>`에서 `T`로 `Vec<u8>`이나 `Stream`등을 사용한다고 했죠? 이 친구는 `T`에 사용하기 위한, 특화된 type입니다. 제일 위의 예시에도 `Request<Body>`를 쓰신 것을 볼 수 있습니다.

### hyper::Response<T>

이전에 봤던 `hyper::Request<T>`와 아주아주 비슷합니다. http response를 감싸는 추상화입니다. 자세한 설명은 생략하겠습니다.

## Client

```rust
#![deny(warnings)]
#![warn(rust_2018_idioms)]
use std::env;

use hyper::{body::HttpBody as _, Client};
use tokio::io::{self, AsyncWriteExt as _};

type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;

#[tokio::main]
async fn main() -> Result<()> {
    pretty_env_logger::init();

    // 사용자가 env로 url 주소를 날리면 그 주소를 받습니다.
    let url = match env::args().nth(1) {
        Some(url) => url,
        None => {
            println!("Usage: client <url>");
            return Ok(());
        }
    };

    // 방금 받은 주소를 parsing 합니다.
    // https는 TLS 연결이 필요한데, 이 예제는 TLS 연결이 구현이 안돼 있어서 http로만 통신이 가능하다네요.
    let url = url.parse::<hyper::Uri>().unwrap();
    if url.scheme_str() != Some("http") {
        println!("This example only works with 'http' URLs.");
        return Ok(());
    }

    fetch_url(url).await
}

async fn fetch_url(url: hyper::Uri) -> Result<()> {
    let client = Client::new();

    // 전달받은 주소로 get request를 날리고, 결과를 받아 옵니다.
    let mut res = client.get(url).await?;

    println!("Response: {}", res.status());
    println!("Headers: {:#?}\n", res.headers());

    // 받아온 response를 그대로 출력합니다.
    while let Some(next) = res.data().await {
        let chunk = next?;
        io::stdout().write_all(&chunk).await?;
    }

    println!("\n\nDone!");

    Ok(())
}

```

### hyper::Client

http request를 날릴 수 있는 `Client` struct입니다. `clone`이 아주 싼 연산이니 멀티쓰레딩 상황에선 `clone`을 사용하라고 권장하고 있습니다.

```rust
use std::time::Duration;
use hyper::Client;

let client = Client::builder()
    .pool_idle_timeout(Duration::from_secs(30))
    .http2_only(true)
    .build_http();

client.get(Uri::from_static("http://httpbin.org/ip")).await?;
```

간단한 `Client`를 만들고 특정 주소로 get request를 보내는 예제입니다.
