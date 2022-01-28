# w4 정리

# Application Layer Protocols

### app layer protocol을 implementation시 생각해야하는 점

- broadcast인가 point-to-point 인가? 앞은 UDP, 뒤는 TCP, UDP다 가능
- reliable transport가 필요한가? 필요하면 TCP만 가능.
- bytestream이 필요한가?(TCP) 아니면 packet-by-packet basis(UDP)도 가능한가?
- 두 party 사이에 end of input을 어떻게 알려주나?
- data format과 encoding은 어떤게 사용되나?



### 많이 쓰이는 protocol 예시

- HTTP와 DNS가 대표적인 app layer protocol
- microservice-based로 가장 유명한 구조가 gRPC
- SMTP : 메일



### 챕터  선 요약

- RPC 특히 gRPC가 어떻게 작동하는지 배우고 간단한 server, client 작성
- email 보내는 lettre 
- FTP client 그리고 TFTP server



# Introduction to RPC

- client가 network를 통해서 procedure call을 할 필요가 늘어나면서 생김.
- RPC (Remote Procedure Call)
- google이 만든 gRPC가 가장 유명하다.
- gRPC는 protocol buffer를 쓴다. 
- message는 HTTP/2, TCP/IP로 전송

# 최근 버전들

### ref

https://medium.com/geekculture/quick-start-to-grpc-using-rust-c655785fc6f4

### grpc_example

- uber같은 프로그램을 만든다.
- client는 name과 location을 가지고, client가 server에 요청하면 server가 근처에 있는 cab들을 알려준다.
- 이전버전으로 고정해야지만 돌아감(뒤에서 확인)

```bash
.
├── Cargo.lock
├── Cargo.toml
├── build.rs
├── proto
│   └── hello.proto
└── src
    ├── client.rs
    ├── lib.rs
    └── server.rs
```



### Cargo.toml

```toml
[package]
name = "learning-grpc"
version = "0.1.0"
authors = ["qubit-finance <lex.studium.12@gmail.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[lib]
path = "./src/lib.rs"

[[bin]]
name="server"
path="./src/server.rs"

[[bin]]
name="client"
path="./src/client.rs"

[dependencies]
tonic = "0.5"
tokio = { version = "1.0", features = ["macros", "rt-multi-thread"] }
prost = "0.8"

[build-dependencies]
tonic-build = "0.5"

```



### hello.proto

### protobuf 설치

```bash
curl -Lo https://github.com/protocolbuffers/protobuf/releases/download/v3.19.3/protoc-3.19.3-linux-x86_64.zip
```

### proto

- 시작은 IDP spec을 명시한다.

- ```protobuf
  syntax = "proto3";
  ```

- package를 선언한다.

- ```proto
  package Hello;
  ```



- Service를 정의하고, rpc 함수를 정의한다.

- ```protobuf
  service Hello {
      rpc HelloWorld(HelloRequest) returns (HelloResponse) {}
      rpc record_cab_location(CabLocationRequest) returns (CabLocationResponse);
      rpc get_cabs(GetCabRequest) returns (GetCabResponse);
  }
  ```

  

- 관련된 message를 정의한다. 여기서 data 종류랑 개수를 명시한다.

```protobuf
syntax = "proto3";

package Hello;

service Hello {
    rpc HelloWorld(HelloRequest) returns (HelloResponse) {}
    rpc record_cab_location(CabLocationRequest) returns (CabLocationResponse);
    rpc get_cabs(GetCabRequest) returns (GetCabResponse);
}

message HelloRequest {}
message HelloResponse {
    string message = 1;
}
message CabLocationRequest {
    string name = 1;
    Location location = 2;
}

message CabLocationResponse {
    bool accepted = 1;
}

message GetCabRequest {
    Location location = 1;
}

message GetCabResponse {
    repeated Cab cabs = 1;
}

message Cab {
    string name = 1;
    Location location = 2;
}

message Location {
    float latitude = 1;
    float longitude = 2;
}

```



- ![](https://miro.medium.com/max/271/1*nLSuqzpIfhYagfu97iFEAQ.png)



### build.rs

```rust
fn main() {
    let proto_file = "./proto/hello.proto"; 

    tonic_build::configure()
        .build_server(true)
        .compile(&[proto_file], &["."])
        .unwrap_or_else(|e| panic!("protobuf compile error: {}", e));
  
        println!("cargo:rerun-if-changed={}", proto_file);
}
```



## tonic 문법 기초

### Response

```rust
let response = HelloResponse { message: "Hello, World!".to_string() };
        Ok(Response::new(response))
```

- 그냥 struct 만들고 new 하면된다.

### Request

```rust
let req = req.into_inner(); // private 접근 가능해짐.
let location = req.location.unwrap();
```

- message는 Some으로 감싸져 있음 (Option임)

### struct

```rust
let location = Location{longitude: 70.1, latitude: 40.1};
let request = Request::new(CabLocationRequest{name: "hi".to_string(), location: Some(location)});
```

- message는 Some으로 감싸줘야한다.

### repeated

```rust
let one = Cab{name: "Limo".to_string(), location: Some(location.clone())};
let two = Cab{name: "Merc".to_string(), location: Some(location.clone())};
let vec = vec![one, two];
let response = GetCabResponse { cabs: vec };
```

- repeated는 그냥 vec으로 주면 된다.



## tonic service 구현

### impl

```rust
#[derive(Default)]
pub struct MyServer {}

#[tonic::async_trait]
impl Hello for MyServer {
    async fn hello_world(&self, _ : Request<HelloRequest>) -> Result<Response<HelloResponse>, Status> {
        let response = HelloResponse { message: "Hello, World!".to_string() };
        Ok(Response::new(response))
    }
}
```

- rpc 구조에 맞게 request랑 response안에 struct 를 넣어주면 된다.




## server.rs

- tonic과
- proto로 만든 녀석들을 가져온다.

```rust
use tonic::transport::Server;
use tonic::{Request, Response, Status};

use learning_grpc::hello; // proto
use hello::hello_server::{HelloServer, Hello}; // service
use hello::{HelloRequest, HelloResponse};
```



#### main

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "0.0.0.0:50051".parse()?;

    let hello_server = MyServer::default();
    Server::builder()
        .add_service(HelloServer::new(hello_server))
        .serve(addr)
        .await?;

    Ok(())
}
```

- macro를 이용해서 바로 runtime에 진입

- builder()->Self

- ```rust
  pub fn add_service<S>(&mut self, svc: S) -> Router<S, Unimplemented, L>
  where
      S: Service<Request<Body>, Response = Response<BoxBody>> + NamedService + Clone + Send + 'static,
      S::Future: Send + 'static,
      S::Error: Into<Box<dyn Error + Send + Sync>> + Send,
      L: Clone, 
  // Create a router with the S typed service as the first service.
  
  //This will clone the Server builder and create a router that will route around different services.
  
  ```





```rust
use tonic::transport::Server;
use tonic::{Request, Response, Status};

use learning_grpc::hello;
use hello::hello_server::{HelloServer, Hello};
use hello::{HelloRequest, HelloResponse, CabLocationRequest, CabLocationResponse,
    GetCabRequest, GetCabResponse, Cab, Location};


#[derive(Default)]
pub struct MyServer {}

#[tonic::async_trait]
impl Hello for MyServer {
    async fn hello_world(&self, _ : Request<HelloRequest>) -> Result<Response<HelloResponse>, Status> {
        let response = HelloResponse { message: "Hello, World!".to_string() };
        Ok(Response::new(response))
    }
    async fn record_cab_location(&self, req : Request<CabLocationRequest>) -> Result<Response<CabLocationResponse>, Status> {
        let response = CabLocationResponse { 
                        accepted: true};
        let req = req.into_inner();
        let location = req.location.unwrap();
        let latitude = location.latitude;
        let longitude = location.longitude;
        println!("Recorded cab {} at {}, {}", req.name, latitude, longitude);
        Ok(Response::new(response))
    }
    async fn get_cabs(&self, _ : Request<GetCabRequest>) -> Result<Response<GetCabResponse>, Status> {
        
        let location = Location{longitude:70.1, latitude:40.1};
        let one = Cab{name: "Limo".to_string(), location: Some(location.clone())};
        let two = Cab{name: "Merc".to_string(), location: Some(location.clone())};
        let vec = vec![one, two];
        let response = GetCabResponse { cabs: vec };
        Ok(Response::new(response))
    }
    
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "0.0.0.0:50051".parse()?;

    let hello_server = MyServer::default();
    Server::builder()
        .add_service(HelloServer::new(hello_server))
        .serve(addr)
        .await?;

    Ok(())
}
```





### client

```rust
use tonic::transport::Endpoint;
use tonic::Request;

use learning_grpc::hello;
use hello::hello_client::HelloClient;
use hello::{HelloRequest, HelloResponse, CabLocationRequest, CabLocationResponse,
    GetCabRequest, GetCabResponse, Cab, Location};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = Endpoint::from_static("https://127.0.0.1:50051");
    
    let mut client  = HelloClient::connect(addr).await?;
    let request = Request::new(HelloRequest{});
    let response = client.hello_world(request).await?;
    println!("response: {}", response.into_inner().message);

    let location = Location{longitude: 70.1, latitude: 40.1};
    let request = Request::new(CabLocationRequest{name: "hi".to_string(), location: Some(location.clone())});
    let response = client.record_cab_location(request).await?;
    println!("response: {}", response.into_inner().accepted);

    let request = Request::new(GetCabRequest{location: Some(location.clone())});
    let response = client.get_cabs(request).await?;
    println!("response: {:?}", response.into_inner().cabs);
    Ok(())
}
```



```bash
response: Hello, World!
response: true
response: [Cab { name: "Limo", location: Some(Location { latitude: 40.1, longitude: 70.1 }) }, Cab { name: "Merc", location: Some(Location { latitude: 40.1, longitude: 70.1 }) }]
```



```bash
Recorded cab hi at 40.1, 70.1
```





# 책의 내용



### grpc_example

- uber같은 프로그램을 만든다.
- client는 name과 location을 가지고, client가 server에 요청하면 server가 근처에 있는 cab들을 알려준다.
- 이전버전으로 고정해야지만 돌아감

### Cargo.toml

```toml
[package]
name = "grpc_example"
version = "0.1.0"
authors = ["Foo<foo@bar.com>"]

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
protobuf = "1.4.1"  # 3.0.0
grpc = "0.2.1" # "0.8.3"
tls-api = "0.1.8" # "0.7.0"

[build-dependencies]
protoc-rust-grpc = "0.2.1" # 0.8.3
```



### build script

```rust
extern crate protoc_rust_grpc;

fn main(){
    protoc_rust_grpc::run(protoc_rust_grpc::Args{
        out_dir: "src",
        includes: &[],
        input: &["foobar.proto"],
        rust_protobuf: true,
    }).expect("Failed to generate Rust arc");
}
```

- foobar.proto 에서 코드를 생성한다. 





### lib.rs

```rust
extern crate protobuf;
extern crate grpc;
extern crate tls_api;

pub mod foobar;
pub mod foobar_grpc;
```





### server

```rust
extern crate grpc_example;
extern crate grpc;
extern crate protobuf;

use std::thread;

use grpc_example::foobar_grpc::*;
use grpc_example::foobar::*;

struct FooBarServer;

// RPC 함수 구현
impl FooBarService for FooBarServer {
    fn record_cab_location(&self,
                _m: grpc::RequestOptions,
                req: CabLocationRequest)
                -> 
    grpc::SingleResponse<CabLocationResponse>{

        let mut r = CabLocationResponse::new();
        
        println!("Recorded cab {} at {}, {}", req.get_name(),
        req.get_location().lattitude, req.get_location().longitude);

        r.set_accepted(true);
        grpc::SingleResponse::completed(r)

    }

    fn get_cabs(&self, 
        _m: grpc::RequestOptions,
        _req: GetCabsRequest)
        -> grpc::SingleResponse<GetCabResponse>
    {
        let mut r = GetCabResponse::new();

        let mut location = Location::new();
        location.lattitude = 40.7128;
        location.longitude = -74.0060;

        let mut one = Cab::new();
        one.set_name("Limo".to_owned());
        one.set_location(location.clone());

        let mut two = Cab::new();
        two.set_name("Merc".to_owned());
        two.set_location(location.clone());

        let vec = vec![one, two];
        let cabs = ::protobuf::RepeatedField::from_vec(vec);

        r.set_cabs(cabs);

        grpc::SingleResponse::completed(r)
    }
}




fn main(){
    let mut server = grpc::ServerBuilder::new_plain();
    server.http.set_port(9001);
    server.add_service(FooBarServiceServer::new_service_def(FooBarServer));
    server.http.set_cpu_pool_threads(4);
    let _server = server.build().expect("Could not start server");
    loop {
        thread::park();
    }
}
```



### client

```rust
extern crate grpc_example;
extern crate grpc;

use grpc_example::foobar::*;
use grpc_example::foobar_grpc::*;


fn main() {
    let client = FooBarServiceClient::new_plain("127.0.0.1", 9001, Default::default()).unwrap();

    let mut req = CabLocationRequest::new();
    req.set_name("foo".to_string());

    let mut location = Location::new();
    location.latitude = 40.730610;
    location.longitude = -73.935242;
    req.set_location(location);

    let resp = client.record_cab_location(grpc::RequestOptions::new(), req);
    match resp.wait() {
        Err(e) => panic!("{:?}", e),
        Ok((_, r, _)) => println!("{:?}", r),
    }

    let mut nearby_req = GetCabRequest::new();
    let mut location = Location::new();
    location.latitude = 40.730610;
    location.longitude = -73.935242;    
    nearby_req.set_location(location);

    let nearby_resp = client.get_cabs(grpc::RequestOptions::new(), nearby_req);
    match nearby_resp.wait() {
        Err(e) => panic!("{:?}", e),
        Ok((_, cabs, _)) => println!("{:?}", cabs),
    }
}

```





