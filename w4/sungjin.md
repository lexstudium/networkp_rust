# Network Protocol

## SMTP

---

SMTP란 `Simple Mail Transfer Protocol`의 줄임말로 OSI 7계층 중 Application layer의 protocol 중 하나이다. 포트번호로 주로 25번을 사용하고 587번도 사용한다.  
러스트에서 SMTP 프로토콜을 사용하여 메일을 보내는 예제를 살펴보자.

### Cargo.toml

```rust
[package]
name = "lettre-example"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
lettre = "0.9.2"
lettre_email = "0.9.2"
```

책에 나온대로 예제를 돌리고 싶지만 쉽지않다..
현재 lettre crate는 최신 버전이 0.10.0-rc4, 출시된지 4년도 지난 책에서 나온 crate가 아직도 정식출시 전인 것 같다.  
책의 `SendableEmail trait`은 최근의 버전에서는 더이상 `trait`이 아니라 `struct`라는 점 등의 이유로
책의 예제들이 돌아가지 않아 간단하게 crate 사용방법을 소개하겠다.  
책과 다르게 `lettre` 와 `lettre_email`을 사용한다.

```rust
use lettre::smtp::authentication::IntoCredentials;
use lettre::{SmtpClient, Transport};
use lettre_email::EmailBuilder;

fn main() {
    let smtp_address = "smtp.gmail.com";
    let username = /* Your ID */;
    let password = /* Your Password */;
    let email = EmailBuilder::new()
        .to(/* Receiver ID */)
        .from(username)
        .subject("메일 테스트")
        .text("lettre로 메일 테스트")
        .build()
        .unwrap()
        .into();
    let credentials = (username, password).into_credentials();
    let mut client = SmtpClient::new_simple(smtp_address)
        .unwrap()
        .credentials(credentials)
        .transport();
    let _result = client.send(email).unwrap();
    println!("{:?}", _result);
}

```

username에 자신의 아이디와 password에 자신의 비밀번호를 적고,  
email 변수 선언 부분에 `EmailBuilder struct`를 계속 사용하는데, `to()` 부분에 상대방의 이메일 주소를 적고 subject에는 메일의 제목, text에는 메일의 내용을 적으면 lettre crate를 사용해서 메일을 전송할 수 있다.  
추가적으로 파일 전송등도 가능하다.  
_gmail을 사용하고 싶은 경우 계정 -> 보안 -> 보안 수준이 낮은 앱의 액세스를 차단에서 사용으로 바꿔야 사용 가능하다. 2차 비밀번호를 사용하는 경우 계정의 비밀번호가 아니라 추가로 설정한 비밀번호로 사용 가능하다._  
[lettre crate doc](https://docs.rs/lettre/latest/lettre/)  
[lettre_email crate doc](https://docs.rs/lettre_email/latest/lettre_email/struct.EmailBuilder.html)

## FTP

---

FTP란 `File Transfer Protocol`의 약자로 파일 전송을 하기 위한 프로토콜이다. `FTP`는 포트번호 20번과 21번을 사용한다.  
[ftp.daum.net - ftp 미러 카카오 예시](ftp.daum.net)  
해당 예시는 책의 예시와 동일하다.

```rust
# Cargo.toml
[package]
name = "ftp-example"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

[dependencies.ftp]
version = "2.2.1"
```

```rust
use ftp::{FtpError, FtpStream};
use std::io::Cursor;
use std::str;

fn run_ftp(addr: &str, user: &str, pass: &str) -> Result<(), FtpError> {
    let mut ftp_stream = FtpStream::connect((addr, 21))?;
    ftp_stream.login(user, pass)?;
    println!("current dir: {}", ftp_stream.pwd()?);

    let data = "A random string to write to a file";
    let mut reader = Cursor::new(data);
    ftp_stream.put("my_file.txt", &mut reader)?;

    ftp_stream.quit()
}
fn main() {
    run_ftp("ftp.dlptest.com", "dlpuser", "rNrKYTX9g7z3RgJRmxWuGHbeu").unwrap();
}

```

코드 자체는 https://dlptest.com/ftp-test/ 에 나와있는 `id`와 `password`에 나와있는것으로 교체하면 실행이 가능하다.  
21번 port로 주어진 아이디와 비밀번호로 접속을 시도하고, 현재 디렉토리를 출력한다.  
`std::io::Cursor`는 인메모리 버퍼를 wrap해서 `Seek`을 제공해준다. 즉, reader나 writer가 자유롭게 I/O가 이뤄지는 위치를 바꿀 수 있게 도와준다.

결과를 확인하기 힘들어 직접 `ftp`서버를 구축해서 실행시켜봤다.

```rust
# Cargo.toml
# ftp 최신버전
[package]
name = "ftp-example"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
ftp = "3.0.1"
```

```rust
use ftp::FtpStream;
use std::io::Cursor;
use std::str;
fn main() {
    let mut ftp_stream = FtpStream::connect("192.168.219.107:21").unwrap();
    let _ = ftp_stream.login("test", "3519").unwrap();

    println!("Current directory: {}", ftp_stream.pwd().unwrap());

    ftp_stream.cwd("./testplace").unwrap();

    let remote_file = ftp_stream.simple_retr("my_upload.txt").unwrap();
    println!(
        "Read file with contents\n{}\n",
        str::from_utf8(&remote_file.into_inner()).unwrap()
    );

    let data = "러스트에서 기록해서 ftp file 생성하기";
    let mut reader = Cursor::new(data);
    ftp_stream.put("my_write.txt", &mut reader).unwrap();

    let remote_file = ftp_stream.simple_retr("my_write.txt").unwrap();
    println!(
        "Read file with contents\n{}\n",
        str::from_utf8(&remote_file.into_inner()).unwrap()
    );

    let _ = ftp_stream.quit();
}

```

`ftp`서버에 `testplace`라는 디렉토리와 `my_upload`라는 파일을 미리 생성해두었다.  
먼저 업로드 해둔 `my_upload` 파일을 `simple_retr()` 함수를 사용해서 찾을 수 있고 읽은 데이터를  
`into_inner()`를 사용해서 가져온다. `into_inner()`는 wrapped value를 얻어 올 수 있다.  
현재 `remote_file`의 type은 `Cursor<Vec<u8>>` 이므로 `into_inner()`을 통해 `Vec<u8>`을 얻는다.  
그 후 `my_write.txt`라는 파일을 `ftp`서버에 업로드 하고, 제대로 업로드 되었는지 다시한번 업로드 된 파일을 다운받아 확인한다.

```
$ cargo run
   Compiling ftp-example v0.1.0 (C:\Users\cho\Desktop\CSE\Rust\ftp-example)
    Finished dev [unoptimized + debuginfo] target(s) in 1.46s
     Running `target\debug\ftp-example.exe`
Current directory: /
Read file with contents
FTP서버에 저장해놓은 메모 파일입니다.

Read file with contents
러스트에서 기록해서 ftp file 생성하기
```

## TFTP

---

TFTP란 `Trivial File Transfer Protocol`의 약자로 `FTP`와 같이 파일을 전송하기 위한 프로토콜이지만, `FTP`보다 더 단순한 방식으로 파일을 전송한다. 데이터 전송 과정에서 데이터가 손상될 수 있는 등 불안한 점이 있지만 구현이 간단하다고 한다. 포트번호로 69번을 사용한다.  
책에 소개된 코드가 잘 되지 않아 수정한 코드이다.

```rust
# Cargo.toml
[package]
name = "tftp-example"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
tftp_server = "0.0.3"
```

```rust
use std::net::SocketAddr;
use std::str::FromStr;

use tftp_server::server::TftpServerBuilder;
fn main() {
    let addr = format!("0.0.0.0:{}", 69);
    let socket_addr = SocketAddr::from_str(addr.as_str()).expect("Error parsing address");
    let builder = TftpServerBuilder::new().addr(socket_addr);
    let mut server = builder.build().unwrap();
    match server.run() {
        Ok(_) => println!("Server completed succesfully!"),
        Err(e) => println!("Failed. Error: {:?}", e),
    }
}
```

버전이 바뀌면서 `TftpServer::new_from_addr()` 로 서버를 생성하지 않고 `TftpServerBuilder::new().adder().build()`를 통해 서버를 생성한다.