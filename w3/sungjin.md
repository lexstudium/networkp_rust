# 4. Data Serialization and Deserialization and parsing

## Serialization and Deserialization

### Serialization   
***
TCP와 UDP 프로토콜은 항상 byte단위로만 데이터를 주고 받는다.  
이전 단원에서는 string에 `as_bytes()`를 붙여 소켓을 통해 전송하였다.  
이러한 행위를 **serialization** 이라고 한다.  

### Deserialization
***
Serialization의 역과정으로 raw data format에서 data structure구조로 담아내는것을 의미한다.
아무래도 serialization은 특정 데이터 포맷으로만 변환하면 되지만 Deserialization과정은 그렇지 않다.  
처음에 delimiter등으로 parsing하나 했는데 그렇지 않다.  
[json도 program source code처럼 parsing한다.](https://www.json.org/json-en.html)


### Serialization과 Deserialization이 필요한 이유
***
Serialization과 Deserialization은 메모리에 있는 데이터를 네트워크 통신에 사용하기 위한 형식으로 바꾸는 과정이다.  
메모리의 Stack이 아니라 Heap영역에 저장되는 데이터의 경우 해당 구조체등 데이터 구조가 Heap영역에 존재하고 Stack메모리에 있는 정보는 Heap영역에 관한 정보를 가지고 있다. 따라서 통신 하기 위해서는 Heap영역에 적힌 data를 가져와서 데이터를 전송해야만, 올바르게 데이터 송수신이 가능하다.  
따라서 올바른 데이터를 가져와 전송하기 위한 수단으로 Serialization이 사용되고, 그 역과정으로 바이트를 읽어내는 과정이 Deserialization 이다.

## Using Serde
[serde](https://serde.rs/)는 Rust에서 거의 표준으로 취급받는 data serializing and deserializing framework이다.  
* 지원하는 Data Format은 다음과 같다.
<img width="791" alt="image" src="https://user-images.githubusercontent.com/86606309/149725874-00e019f5-71f6-4142-b2f7-96d5325dc6e6.png">

* 지원하는 Data Structures들은 `String, &str, usize, Vec<T>, HashMap<K, V>` 등의 러스트의 common data structure등을 지원한다.
---
### Serde-basic
```rust
use serde;
use serde_json;
use serde_yaml;
use serde_derive::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct ServerConfig {
    workers:        u64,
    ignore:         bool,
    auth_server:    Option<String>
}

fn main() {
    let config = ServerConfig { workers: 100, ignore: false, auth_server: Some("auth.server.io".to_string()) };
    {
        println!("To and from YAML");
        let serialized = serde_yaml::to_string(&config).unwrap();
        println!("{}", serialized);
        let deserialized: ServerConfig = serde_yaml::from_str(&serialized).unwrap();
        println!("{:?}", deserialized);
    }
    println!("\n\n");
    {
        println!("To and from JSON");
        let serialized = serde_json::to_string(&config).unwrap();
        println!("{}", serialized);
        // for i in serialized.as_bytes() {
        //     println!("bytes: {} char: {}", i, *i as char);
        // }
        let deserialized: ServerConfig = serde_json::from_str(&serialized).unwrap();
        println!("{:?}", deserialized);
    }
}
```
위 예시는 하나의 구조체를 선언하고 Serde framework를 사용하여 Serialize와 Deserialize를 한 코드이다.
`serde_{format}::to_string()` 을 사용하여 Serialize할 수 있고  
`serde_{format}::from_str()`을 사용하여 Deserialize할 수 있다.  
`serde_derive` crate에서 `Serialization과 Deserialization trait`을 가지고 있으며 해당 기능을 수행하고자 하는 Data structure에 trait을 추가해주면 사용 가능하다.  

---
### Serde-server
```rust
use serde_json;
use serde_derive::{Serialize, Deserialize};

use std::net::{TcpListener, TcpStream};
use std::io::{stdin, BufRead, BufReader, Error, Write};
use std::{env, str, thread};

#[derive(Serialize, Deserialize, Debug)]
struct Point3D {
    x: u32,
    y: u32,
    z: u32,
}

fn handle_client(stream: TcpStream) -> Result<(), Error> {
    println!("Incoming connection from: {}", stream.peer_addr()?);
    let mut data = Vec::new();
    let mut stream = BufReader::new(stream);

    loop {
        data.clear();

        let bytes_read = stream.read_until(b'\n', &mut data)?;
        if bytes_read == 0 {
            return Ok(());
        }
        let input: Point3D = serde_json::from_slice(&data)?;
        let value = input.x.pow(2) + input.y.pow(2) + input.z.pow(2);

        write!(stream.get_mut(), "{}", f64::from(value).sqrt())?;
        write!(stream.get_mut(), "{}", "\n")?;
    }
}

fn main() {
    let args: Vec<_> = env::args().collect();
    if args.len() != 2 {
        eprintln!("Please provide --client or --server as argument");
        std::process::exit(1);
    }
    if args[1] == "--server" {
        let listener = TcpListener::bind("0.0.0.0:8888").expect("Could not bind");
        for stream in listener.incoming() {
            match stream {
                Err(e) => eprintln!("failed: {}", e),
                Ok(stream) => {
                    thread::spawn(move || {
                        handle_client(stream).unwrap_or_else(|error| eprintln!("{:?}", error));
                    });
                }
            }
        }
    } else if args[1] == "--client" {
        let mut stream = TcpStream::connect("127.0.0.1:8888").expect("Could not connect to server");
        println!("Please provide a 3D point as three comma separated integers");
        loop {
            let mut input = String::new();
            let mut buffer: Vec<u8> = Vec::new();
            stdin()
                .read_line(&mut input)
                .expect("Failed to read from stdin");
            let parts: Vec<&str> = input.trim_matches('\n').split(',').collect();
            let point = Point3D {
                x: parts[0].parse().unwrap(),
                y: parts[1].parse().unwrap(),
                z: parts[2].parse().unwrap(),
            };
            stream
                .write_all(serde_json::to_string(&point).unwrap().as_bytes())
                .expect("Failed to write to server");
            stream.write_all(b"\n").expect("Failed to write to server");

            let mut reader = BufReader::new(&stream);
            reader
                .read_until(b'\n', &mut buffer)
                .expect("Could not read into buffer");
            let input = str::from_utf8(&buffer).expect("Could not write buffer as string");
            if input == "" {
                eprintln!("Empty response from server");
            }
            print!("Response from server {}", input);
        }
    }
}
```
이전장의 TCP 서버-클라이언트 간 통신과 같으나 주고받는 데이터가 String에서 Point3D 구조체와 그 거리로 바뀐 예제이다.

실행 결과는 다음과 같다.
<img width="1606" alt="image" src="https://user-images.githubusercontent.com/86606309/149730623-2913e2df-a192-4b9a-b825-0eed94754ce9.png">

서버에서는 클라이언트로부터 Point3D 구조체의 Serialize된 형태로 전달한다. 그러면 서버에서 Point3D 구조체를 Deserialize를 통해 얻고, 좌표 3개를 통하여 거리를 계산해 다시 stream에 String의 형태로 적어 클라이언트로 전송하면 클라이언트는 거리에 관한 정보를 얻는다.