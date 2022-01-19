# 네트워크와 관련된 외부 크레이트들

cargo에 dependency 추가하는 방법은 다들 아실 거라고 믿고 넘어가겠습니다~

## ipnetwork

[github](https://github.com/achanda/ipnetwork)

ip주소로 CIDR와 관련된 연산을 도와주는 라이브러리입니다.

[CIDR](https://kim-dragon.tistory.com/9)

CIDR가 뭔지 간략하게 설명을 드리자면, 32bit의 IP 주소를 두 부분으로 나눠서 앞부분으로는 network 주소, 뒷부분으로는 sub-net 주소로 사용하는 방식이라네요.
예를 들어서 A라는 네트워크에 16명의 참여자가 있고, CIDR 값이 28이라면, 앞의 28bit는 네트워크 이름(여기선 A)을 나타내고, 뒤의 4bit로 참여자를 식별한다는 방식인 거 같습니다. 제가 맞게 이해한 건지 모르겠네요.ㅎㅎ

```rust

// Rust2018부터는 `extern crate` 쓰지 않아도 됩니다.
//extern crate ipnetwork;

use std::net::Ipv4Addr;
use ipnetwork::{IpNetwork, Ipv4Network, Ipv6Network};

fn main() {

    // `net`과 `str_net`은 동일한 값을 다른 방식으로 표현했습니다.
    // 192.168.22.0이라는 IP 주소의 뒷 10bit를 sub-net으로 사용하겠다는 뜻입니다.
    let net = IpNetwork::new("192.168.122.0".parse().unwrap(), 22)
        .expect("Could not construct a network");

    let str_net: IpNetwork = "192.168.122.0/22".parse().unwrap();

    assert!(net == str_net);
    assert!(net.is_ipv4());

    let net4: Ipv4Network = "192.168.121.0/22".parse().unwrap();

    // sub-net의 크기가 10bit가 맞는지 확인합니다.
    assert!(net4.size() == 2u64.pow(32 - 22));

    // 192.168.121.0에서 시작해서 10bit이기 때문에 192.168.121.3은 범위에 포함됩니다.
    assert!(net4.contains(Ipv4Addr::new(192, 168, 121, 3)));

    let _net6: Ipv6Network = "2001:db8::0/96".parse().unwrap();

    for addr in net4.iter().take(10) {
        println!("{}", addr);
    }

}
```

## mio

[github](https://github.com/tokio-rs/mio)

이전까지는 stream과 buffer를 이용해서 원시적으로 통신을 구현했다면, mio는 이벤트 루프를 사용할 수 있게 도와줍니다.

```rust
use std::error::Error;

use mio::net::{TcpListener, TcpStream};
use mio::{Events, Interest, Poll, Token};

// 식별자
const SERVER: Token = Token(0);
const CLIENT: Token = Token(1);

fn main() -> Result<(), Box<dyn Error>> {

    // 이벤트를 받는 친구
    let mut poll = Poll::new()?;

    // 최대 128개까지 이벤트를 받을 수 있습니다.
    let mut events = Events::with_capacity(128);


    // Tcp 하나를 열고 걔를 poll에 연결하여, 이벤트를 줄 수 있도록 합니다.
    // 이 친구가 주는 이벤트들에는 `SERVER`라는 토큰이 붙어 있습니다.
    let addr = "127.0.0.1:13265".parse()?;
    let mut server = TcpListener::bind(addr)?;
    poll.registry()
        .register(&mut server, SERVER, Interest::READABLE)?;

    // 클라이언트도 서버와 동일하게 해줍니다.
    let mut client = TcpStream::connect(addr)?;
    poll.registry()
        .register(&mut client, CLIENT, Interest::READABLE | Interest::WRITABLE)?;

    // 루프를 돌면서 이벤트들을 처리합니다.
    loop {
        poll.poll(&mut events, None)?;

        for event in events.iter() {
            
            // 토큰을 보고 서버에서 보낸 건지, 클라이언트에서 보낸 건지 구분합니다.
            match event.token() {
                SERVER => {

                    // accept가 뭐하는 친구인지 이해가 안 돼서 원문의 주석을 그대로 남겨놨습니다...
                    // If this is an event for the server, it means a connection
                    // is ready to be accepted.
                    //
                    // Accept the connection and drop it immediately. This will
                    // close the socket and notify the client of the EOF.
                    let connection = server.accept();
                    drop(connection);
                }
                CLIENT => {
                    if event.is_writable() {
                        // 처리 로직
                    }

                    if event.is_readable() {
                        // 처리 로직
                    }

                    // 여기도 무슨 코드인지 이해가 안 가서 그대로 놔뒀습니다.
                    // Since the server just shuts down the connection, let's
                    // just exit from our event loop.
                    return Ok(());
                }

                // 이렇게 할 거면 그냥 enum을 쓰면 되는데 왜 굳이 Token을 사용할까요...
                _ => unreachable!(),
            }
        }
    }
}
```

## pnet

[github](https://github.com/libpnet/libpnet)

Rust의 standard library에서 제공하는 것보다 훨씬 low-level한 제어가 가능하다고 하네요.

Transport protocol을 직접 구현하거나, 패킷 주고 받는 과정을 정교하게 조작할 수 있다고 하는데, 자세한 내용이 궁금하신 분들은 깃허브 Readme를 읽어주시면 되겠습니다.

아래는 책의 예시입니다. arguments로 인터페이스를 설정하고, 받아온 패킷들을 `handle_packet`에서 처리합니다.

```rust
use pnet::datalink::{self, NetworkInterface};
use pnet::datalink::Channel::Ethernet;
use pnet::packet::ethernet::{EtherTypes, EthernetPacket};
use pnet::packet::ipv4::Ipv4Packet;
use pnet::packet::tcp::TcpPacket;
use pnet::packet::ip::IpNextHeaderProtocols;
use pnet::packet::Packet;
use std::env;


// Handles a single ethernet packet
fn handle_packet(ethernet: &EthernetPacket) {

    match ethernet.get_ethertype() {
        EtherTypes::Ipv4 => {
            let header = Ipv4Packet::new(ethernet.payload());

            if let Some(header) = header {

                match header.get_next_level_protocol() {
                    IpNextHeaderProtocols::Tcp => {
                        let tcp = TcpPacket::new(header.payload());

                        if let Some(tcp) = tcp {
                            println!(
                                "Got a TCP packet {}:{} to {}:{}",
                                header.get_source(),
                                tcp.get_source(),
                                header.get_destination(),
                                tcp.get_destination()
                            );
                        }

                    }
                    _ => println!("Ignoring non TCP packet"),
                }
            }
        }
        _ => println!("Ignoring non IPv4 packet"),
    }

}
fn main() {

    let interface_name = env::args().nth(1).unwrap();

    // Get all interfaces
    let interfaces = datalink::interfaces();

    // Filter the list to find the given interface name
    let interface = interfaces
        .into_iter()
        .filter(|iface: &NetworkInterface| iface.name == interface_name)
        .next()
        .expect("Error getting interface");

    let (_tx, mut rx) = match datalink::channel(&interface, Default::default()) {
        Ok(Ethernet(tx, rx)) => (tx, rx),
        Ok(_) => panic!("Unhandled channel type"),
        Err(e) => {
            panic!("An error occurred when creating the datalink channel: {}",e)
        }
    };

    // Loop over packets arriving on the given interface
    loop {

        match rx.next() {
            Ok(packet) => {
                let packet = EthernetPacket::new(packet).unwrap();
                handle_packet(&packet);
            }
            Err(e) => {
                panic!("An error occurred while reading: {}", e);
            }
        }

    }

}
```

...그렇다네요

## trust-dns

[github](https://github.com/bluejekyll/trust-dns)

Rust std의 DNS 관련 기능이 빈약한데, 그걸 보완해주는 크레이트입니다.

DNS가 뭔지 모르시는 분들은 [2018학년도 6월 모의고사 국어영역](https://www.suneung.re.kr/boardCnts/view.do?boardID=1500236&boardSeq=5013300&lev=0&m=0403&searchType=null&statusYN=W&page=9&s=suneung)을 찾아보시면 됩니당 ㅎㅎ

간단하게 설명드리자면 `https://www.naver.com`이라는 주소를 `223.130.200.107`라는 ip 주소로 바꿔주는 역할을 하는 친구입니다.

```rust
use std::env;
use trust_dns_resolver::Resolver;
use trust_dns_resolver::config::*;
use trust_dns::rr::record_type::RecordType;

fn main() {

    // 들어가고 싶은 주소를 arguments를 통해서 전달 받습니다.
    let args: Vec<String> = env::args().collect();

    if args.len() != 2 {
        eprintln!("Please provide a name to query");
        std::process::exit(1);
    }

    // 기본으로 제공되는 DNS를 사용합니다. 구글의 DNS 서버를 사용한다는 군요.
    let resolver = Resolver::new(ResolverConfig::default(), ResolverOpts::default()).unwrap();

    // FQDN(Fully Qualified Domain Name)이 되려면 이름 마지막에 `.`을 붙여야합니다.
    let query = format!("{}.", args[1]);

    // 방금 만든 쿼리를 DNS 서버에 보내서 ip를 구해옵니다.
    let response = resolver.lookup_ip(query.as_str());

    println!("Using the synchronous resolver");

    for ans in response.iter() {
        println!("{:?}", ans);
    }

    println!("Using the system resolver");

    // system에 있는 `resolv.conf`에서 결과를 찾는 Resolver입니다.
    let system_resolver = Resolver::from_system_conf().unwrap();
    let system_response = system_resolver.lookup_ip(query.as_str());

    for ans in system_response.iter() {
        println!("{:?}", ans);
    }

    let ns = resolver.lookup(query.as_str(), RecordType::NS);

    println!("NS records using the synchronous resolver");

    for ans in ns.iter() {
        println!("{:?}", ans);
    }

}
```
