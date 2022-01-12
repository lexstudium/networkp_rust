
# A Simple UDP server and client

UDP는 TCP와는 달리 `stream 구조체`를 사용하지 않는다.
UDP echo server는 어떻게 생겼는지 살펴보자.

## UDP echo server

```rust
use std::thread;
use std::net::UdpSocket;

fn main() {

    let socket = UdpSocket::bind("0.0.0.0:8888").expect("Could not bind socket");

    loop {
        let mut buf = [0u8; 1500];
        let sock = socket.try_clone().expect("Failed to clone socket");
        match socket.recv_from(&mut buf) {
            Ok((_, src)) => {
                thread::spawn(move || {
                    println!("Handling connection from {}", src);
                    sock.send_to(&buf, &src).expect("Failed to send a response");
                });
            },
            Err(e) => {
                eprintln!("couldn't recieve a datagram: {}", e);
            }
        }
    }
}
```
TCP 기반으로 만든 echo server의 경우에는 두 hosts 간의 전송 효율을 위해 `sliding window(BufReader)`를 사용했었지만, UDP는 connectionless 프로토콜이기 때문에 필요가 없다. 그래서 고정된 크기의 버퍼에 데이터를 받아오기만 하면 된다. 
> 그 크기는 일반적인 LAN의 `MTU(Maximum Transmission Unit)`인 약 1,500으로 설정한다.

`try_clone()` 함수로 소켓의 클론을 생성하고 이후 스레드가 생성될 때 `move closure` 내부로 ownership이 넘어간다.

`recv_from()` 함수로 데이터를 성공적으로 읽어왔다면 `Ok()` branch로 넘어간 뒤 `send_to()` 함수에서 echo 된다.

서버 실행 후 `nc` 명령어를 통해 접근할 때에는 옵션으로 `-u`를 넣어 UDP 프로토콜이라는 것을 명시한다.

## UDP client

```rust
use std::net::UdpSocket;
use std::{str,io};

fn main() {
    let socket = UdpSocket::bind("127.0.0.1:8000").expect("Could not bind client socket");
    socket.connect("127.0.0.1:8888").expect("Could not connect to server");
    loop {
        let mut input = String::new();
        let mut buffer = [0u8; 1500];
        io::stdin().read_line(&mut input).expect("Failed to read from stdin");
        socket.send(input.as_bytes()).expect("Failed to write to server");

        socket.recv_from(&mut buffer).expect("Could not read into buffer");
        print!("{}", str::from_utf8(&buffer).expect("Could not write buffer as string"));
    }
}
```
UDP client는 TCP client 때와 그렇게 다르지 않다. 차이점이 있다면 UDP client에선 server와 `bind` 해주는 것이 필수다.
> 이유는 단순하게 UDP가 connectionless 프로토콜이기 때문이다. 이렇게 bind 해주지 않으면 모든 다른 프로그램에게 UDP로 뿌려질 것이기 때문이다. [Reference](https://stackoverflow.com/questions/3057029/do-i-have-to-bind-a-udp-socket-in-my-client-program-to-receive-data-i-always-g)


# UDP multicasting

`UdpSocket` type은 TCP type에는 없는 여러 메소드들이 있는데 그 중 가장 흥미로운 것들은 `multicasting`과 `broadcasting`이다. 이 중 multicasting을 이용한 server, client의 예제를 살펴보자.

> Reference - [multicast](https://en.wikipedia.org/wiki/Multicast)란?

```rust
use std::{env, str};
use std::net::{UdpSocket, Ipv4Addr};

fn main() {
    let mcast_group: Ipv4Addr = "239.0.0.1".parse().unwrap();
    let port: u16 = 6000;
    let any = "0.0.0.0".parse().unwrap();
    let mut buffer = [0u8; 1600];
    if env::args().count() > 1 {
	    // client case
        let socket = UdpSocket::bind((any, port)).expect("Could not bind client socket");
        socket.join_multicast_v4(&mcast_group, &any).expect("Could not join multicast group");
        socket.recv_from(&mut buffer).expect("Failed to write to server");
        print!("{}", str::from_utf8(&buffer).expect("Could not write buffer as string"));
    } else {
	    // server case
        let socket = UdpSocket::bind((any, 0)).expect("Could not bind socket");
        socket.send_to("Hello world!".as_bytes(), &(mcast_group, port)).expect("Failed to write data");
    }
}
```
`join_multicast_v4()` [함수](https://doc.rust-lang.org/std/net/struct.UdpSocket.html#method.join_multicast_v4)로 수신받을 호스트들을 지정해준다. `any` 변수가 가진 `0.0.0.0`은 리눅스 소켓 프로그래밍에서 쓰이는 `INADDR_ANY`를 의미한다. 모든 주소를 대변한다고 볼 수 있다. 또한 multicast address를 지정해주고 있는 `mcast_group` 변수가 가진 `239.0.0.1`은 특별한 의미를 가진 address다. 첫 4-bit를 `1110`으로 세팅하면 multicast를 지시하는 것으로 약속되어 있다.

> `239.0.0.1`를 이진수로 나타내면 `1110 1111.0000 0000.0000 0000.0000 0001`이다. multicast를 나타내고 있으며 위의 주소는 사설 multicast 영역을 의미한다. [Reference](https://unabated.tistory.com/entry/Multicast-1-%EA%B8%B0%EB%B3%B8-%EC%9D%B4%EB%A1%A0)

`UdpSocket` type은 이외에도 leaving multicast groups, broadcasting, ... 등을 지원한다.

> TCP는 당연히 broadcasting 같은 것을 지원하지 않는다.


# Miscellaneous utilities in std::net

## IpAddr, SocketAddr

std::net에 있는 또다른 중요한 type으로는 `IpAddr`이 있다. IP address를 나타낼 때 사용하는데 V4, V6를 variant로 가지는 enum 형태로 존재한다. 비슷한 type으로는 `SocketAddr`이 있다. IP address + port number를 저장한다. 마찬가지로 V4, V6 두 가지 variant가 있다.

```rust
#![feature(ip)]

use std::net::{IpAddr, SocketAddr};

fn main() {
    let local: IpAddr = "127.0.0.1".parse().unwrap();
    assert!(local.is_loopback());

    let global: IpAddr = IpAddr::from([0, 0, 0x1c9, 0, 0, 0xafc8, 0, 0x1]);
    assert!(global.is_global());

    let local_sa: SocketAddr = "127.0.0.1:80".parse().unwrap();
    assert!(local_sa.is_ipv4());

    let global_sa = SocketAddr::new(global, 80u16);
    assert!(global_sa.is_ipv6());
}
```
하지만 위의 type들이 가진 여러 메소드 중 몇몇은 아직 안정화되지 않아서 nightly compiler에서만 이용이 가능하다. 이 글을 쓰는 현재까지도 [issue](https://doc.rust-lang.org/beta/unstable-book/library-features/ip.html)를 보면 여전히 토론 중이라는 것을 알 수 있다. 그래서 사용하기 위해서는 nightly compiler가 필요하고 소스 코드에서 compiler feature flag를 통해 알려줘야 한다.

위의 예제에서는 `is_global()` 함수가 아직 안정화되지 않았기 때문에 feature flag를 사용한 것이다.
> 추가로 어떤 함수가 아직 논의 중인지, 어떤 함수가 추가로 생길 예정인지 궁금하다면 바로 위 issue 링크로 들어가면 보이는 가장 처음 reply가 그 리스트다.

## DNS lookup

`lookup_host()` 함수를 사용해서 DNS service의 결과인 IP address를 알아낼 수 있다. 리턴값으로 `LookupHost` type이 반환된다. `is_global()` 함수와 같이 nightly compiler에서만 동작한다.

```rust
#![feature(lookup_host)]

use std::env;
use std::net::lookup_host;

fn main() {
    let args: Vec<_> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide only one host name");
        std::process::exit(1);
    } else {
        let addresses = lookup_host(&args[1]).unwrap();
        for address in addresses {
	        // ip() method for extracting the IP
            println!("{}", address.ip());
        }
    }
}
```
그리고 reverse DNS lookup 기능은 현재 standard library에서는 [사용 불가능](https://github.com/rust-lang/rust/issues/22608)하다. 그래서 대신에 `trust-dns`와 같은 crate를 사용해서 위의 기능을 이용할 수 있다.
