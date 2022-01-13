# TCP and UDP Using Rust

Rust 의 `std::net` namespace가 network 관련 라이브러리를 지원한다.  
그리고 `Read` 와 `Write` trait은 `std::io`를 불러와 사용할 수 있다.  

## Important structure
Rust [std::net](https://doc.rust-lang.org/std/net/index.html)

* IpAddr - IPv4와 IPv6에 상관없이 사용할 수 있는 IP에 관한 정보를 담은 구조체
``` rust
# 정의
pub enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}

# 예제 
use std::net::{IpAddr, Ipv4Addr, Ipv6Addr};

let localhost_v4 = IpAddr::V4(Ipv4Addr::new(127, 0, 0, 1));
let localhost_v6 = IpAddr::V6(Ipv6Addr::new(0, 0, 0, 0, 0, 0, 0, 1));

assert_eq!("127.0.0.1".parse(), Ok(localhost_v4));
assert_eq!("::1".parse(), Ok(localhost_v6));

assert_eq!(localhost_v4.is_ipv6(), false);
assert_eq!(localhost_v4.is_ipv4(), true);
```


* SocketAddr - 소켓의 주소(IP주소 + 포트번호)를 담은 구조체
``` rust
# 정의
pub enum SocketAddr {
    V4(SocketAddrV4),
    V6(SocketAddrV6),
}

# 예제
use std::net::{IpAddr, Ipv4Addr, SocketAddr};

let socket = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 0, 1)), 8080);

assert_eq!("127.0.0.1:8080".parse(), Ok(socket));
assert_eq!(socket.port(), 8080);
assert_eq!(socket.is_ipv4(), true);
```

* TcpListenr, TcpStream, UdpSocket...
``` 
TcpListner => TCP 서버에서 소켓주소를 binding하고 TCP connection을 기다림  
TcpStream => Server와 Client간 연결 후 데이터가 흐르는 통로  
UdpSocket => 소켓주소를 binding하고 다른 소켓주소와 통신을 가능하게 함
```

현재 네트워크 어플리케이션의 고성능을 위해 tokio 라는 비동기 프로그래밍을 도와주는 라이브러리를 사용하며 tokio는 이후 챕터에서 다뤄질 예정

## TCP Server and client

### Server Code
``` rust
use std::net::{TcpListner, TcpStream};
use std::thread;

use std::io::{Read, Write, Error};

fn handle_client(mut stream: TcpStream) -> Result<(), Error> {
    println!("Incomming connection from: {}", stream.peer_addr()?);
    let mut buf = [0; 512];
    loop {
        let bytes_read = stream.read(&mut buf)?;
        if bytes_read == 0 { return Ok(()); }
        stream.write(&buf[..bytes_read])?;
    }
}

fn main() {
    let listener = TcpListner::bind("0.0.0.0:8888").expect("Could not bind");

    for stream in listener.incoming() {
        match stream {
            Err(e) => { eprintln!("failed: {}", e) },
            Ok(stream) => {
                thread::spawn(move || {
                    handle_client(stream)
                    .unwrap_or_else(|error| eprintln!("{:?}", error));
                })
            }
        }
    }
}
```
`TcpListner`을 통해 Client의 connection을 기다리는 socket을 하나 생성하고, 소켓의 주소를 설정한다.

`incoming` method는 연결된 Server의 stream에 관한 iterator을 반환한다.  

`match`와 `?`를 사용해 상태에 따른 에러처리를 한다. 

`peer_addr` 을 통해 remote connection의 socket address를 얻을 수 있다.

### echo test

nc <server ip> port 로 echo 확인 가능

```
test
test
park
park
^C
```

### Client Code

``` rust
# nc와 동일하게 동작
use std::net::TcpStream;
use std::str;
use std::io::{self, BufRead, BufReader, Write};

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:8888")
    .expect("Could not connect to server");

    loop {
        let mut input = String::new();
        let mut buffer: Vec<u8> = Vec::new();
        io::stdin().read_line(&mut input)
        .expect("Failed to read from stdin");
        stream.write(input.as_bytes()).expect("Failed to write to server");

        let mut reader = BufReader::new(&stream);
        reader.read_until(b'\n', &mut buffer).expect("Could not read into buffer");
        print!("{}", str::from_utf8(&buffer).expect("Could not write buffer as string"));
    }
}
```

Client쪽에서는 `TcpStream::connect`를 사용하여 Server와 연결을 생성한다. 모든 TCP connection에서 client는 remote IP 와 port번호를 알아야 연결할 수 있다.



io::stdin().read_line(&mut input)을 사용하면 standard input을 읽어와 input에 저장할 수 있다.


Rust의 read와 write는 모두 buf의 data type이 u8인 것을 인자로 받으므로 buffer와 input등을 모두 u8이나 byte 단위로 바꿔서 인자로 전달한다. 

``` rust
fn write(&mut self, buf: &[u8]) -> Result<usize>
fn read(&mut self, buf: &mut [u8]) -> Result<usize>
```

그리고 벡터를 통해 string slice type을 얻을 수 있어 원본 문자열을 다시 얻을 수 있다.
``` rust
usd std::str

pub fn from_utf8(v: &[u8]) -> Result<&str, Utf8Error>
```