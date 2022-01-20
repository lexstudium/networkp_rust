# w3 설명자료

## Parsing textual data

- PEG(Parsing Expression Grammar) : parsing 하는 룰,,,
- 여러가지 방법이 있다.



- domain-specific parsing을 정의하기위해 macro를 사용하기
- macro는 항상 디버깅이 어렵다.
- 빠르게 코드 생성이 되긴함.
- overloading이 안되기 때문에 DSL을 정의해야함.
- ex) nom



- trait을 사용하는 방법
- customize가 더 쉽고, debug랑 maintain도 쉽다.
- ex) pom, pest



## pom

- parser를 작은 parser들의 나열로 정의함.
- '*'가 overloading 돼서 sequence를 연결하는 연락을 함.
- 책 1.1.0
- 현재 3.2.0

```rust
extern crate pom;

use pom::DataInput;
use pom::parser::{sym, one_of, seq};
use pom::parser::*;

use std::str;

// space나 개행 찾아서 없애줌
fn space() -> Parser<'static, u8, ()>{
    one_of(b" \t\r\n").repeat(0..).discard()
}

fn string() -> Parser<'static, u8, String>{
    one_of(b"abcdefghijklmnopqrstuvwxyz").repeat(0..)
    .convert(String::from_utf8)
}

fn main() {
    let get = b"GET /home/ HTTP/1.1\r\n";
    let mut input = DataInput::new(get);
    let parser = (seq(b"GET") | seq(b"POST")) * space() * sym(b'/') *
    string() * sym(b'/') * space() * seq(b"HTTP/1.1");
    let output = parser.parse(&mut input);
    println!("{:?}", str::from_utf8(&output.unwrap()));
}

```

- seq는 해당 문자열을 catch 하는 역할, |는 logical or임



### seq

```rust
pub fn seq<'a, 'b: 'a, I>(tag: &'b [I]) -> Parser<'a, I, &'a [I]> 
where
    I: PartialEq + Debug, 
```

- Success when sequence of symbols matches current input.



### Parser

```rust
pub struct Parser<'a, I, O> {
    pub method: Box<dyn Fn(&'a [I], usize) -> Result<(O, usize)> + 'a>,
}

pub fn parse(&self, input: &'a [I]) -> Result<O>
// Apply the parser to parse input.

pub fn repeat<R>(self, range: R) -> Parser<'a, I, Vec<O>>
where
    R: RangeArgument<usize> + Debug + 'a,
    O: 'a, 

// p.repeat(5) repeat p exactly 5 times p.repeat(0..) repeat p zero or more times p.repeat(1..) repeat p one or more times p.repeat(1..4) match p at least 1 and at most 3 times


pub fn convert<U, E, F>(self, f: F) -> Parser<'a, I, U>
where
    F: Fn(O) -> Result<U, E> + 'a,
    E: Debug,
    O: 'a,
    U: 'a, 

// Convert parser result to desired value, fail in case of conversion error.

pub fn discard(self) -> Parser<'a, I, ()>
where
    O: 'a, 

// Discard parser output.
```



### sym

```rust
pub fn sym<'a, I>(t: I) -> Parser<'a, I, I> 
where
    I: Clone + PartialEq + Display, 

// Success when current input symbol equals t
```





### one_of

```rust
pub fn one_of<'a, I, S: ?Sized>(set: &'a S) -> Parser<'a, I, I> 
where
    I: Clone + PartialEq + Display + Debug,
    S: Set<I>, 

// Success when current input symbol is one of the set.
```



## 최근버전 코드

```rust
extern crate pom;

//use pom::DataInput;
use pom::parser::{sym, one_of, seq};
use pom::parser::*;

use std::str;

// space나 개행 찾아서 없애줌
fn space() -> Parser<'static, u8, ()>{
    one_of(b" \t\r\n").repeat(0..).discard()
}

fn string() -> Parser<'static, u8, String>{
    one_of(b"abcdefghijklmnopqrstuvwxyz").repeat(0..).convert(String::from_utf8)
}

fn main() {
    //let get = b"GET /home/ HTTP/1.1\r\n";
    //let mut input = DataInput::new(get);
    let input = b"GET /home/ HTTP/1.1\r\n";
    let parser = (seq(b"GET") | seq(b"POST")) * space() * sym(b'/') *
    string() * sym(b'/') * space() * seq(b"HTTP/1.1");
    let output = parser.parse(input);
    println!("{:?}", str::from_utf8(&output.unwrap()));
}

```



## nom

- small parser 들을 combine해서 더 크고 효과적인 parser를 만드는 구조.
- 최근버전이랑 상당히 다름
- macro_based => function_based
- 책의 버전을 쓰면 책의 코드가 돌지 않음,,,
- 4.1.0 사용

### 매크로 버전 문법

```rust
named!
// 후에 그냥 함수 정의로 대체됨
tag!("GET")
// 정확히 일치하면 그부분을 자름

alt!(f1, f2)
// f1시도하고 실패하면 f2 시도함

return_error!
// 도중에 실패하면 끝까지 안가고 바로 return함

ws!
// white space trim

do_parse!
// sub-parser들을 연결함.

map_res!
// result를 map
```



```rust
#[macro_use]
extern crate nom;

use std::str;
use nom::{
    IResult,
    ErrorKind,
};

use nom::types::{CompleteStr, CompleteByteSlice};

#[derive(Debug)]
enum Method{
    GET,
    POST,
}

#[derive(Debug)]
struct Request {
    method: Method,
    url: String,
    version: String,
}


named!(parse_method<&[u8], Method>, 
    return_error!(ErrorKind::Custom(12), alt!(map!(tag!("GET"), |_| 
    Method::GET) | map!(tag!("POST"), |_| Method::POST))));

named!(parse_request<&[u8], Request>, ws!(do_parse!(
    method: parse_method >>
    url: map_res!(take_until!(" "), str::from_utf8) >> 
    tag!("HTTP/") >> 
    version: map_res!(take_until!("\r"), str::from_utf8) >> 
    (Request { 
        method: method, 
        url: url.into(), 
        version: version.into()
        }
    ))));

    
fn run_parser(input: CompleteStr)
{
    match parse_request(input.as_bytes()) {
        IResult::Ok((rest, value)) => println!("Rest: {:?} Value : {:?}", rest, value),
        IResult::Err(err) => println!("Error: {:?}", err),
        
        
    }
}
fn main() {
    let get = CompleteStr::from("GET /home/ HTTP/ 1.1\r\n");
    run_parser(get);
    let post = CompleteStr::from("POST /update/ HTTP/1.1\r\n");
    run_parser(post);
    let wrong = CompleteStr::from("WRONG /wrong/ HTTP/1.1\r\n");
    run_parser(wrong);
}

```



- Incomplete(Size(1)) 에러가 계속 발생
- 검색결과, 얘네는 string을 stream 으로 계속 받는거라서 string이 언제 끝나는지 모른다고 함,,,
- 



```rust
#[macro_use]
extern crate nom;

use std::str;
use nom::{
    IResult,
    ErrorKind,
};

use nom::types::CompleteStr;

#[derive(Debug)]
enum Method{
    GET,
    POST,
}

#[derive(Debug)]
struct Request {
    method: Method,
    url: String,
    version: String,
}


named!(parse_method<&[u8], Method>, 
    return_error!(ErrorKind::Custom(12), alt!(map!(tag!("GET"), |_| 
    Method::GET) | map!(tag!("POST"), |_| Method::POST))));

named!(parse_request<&[u8], Request>, ws!(do_parse!(
    method: parse_method >>
    url: map_res!(take_until!(" "), str::from_utf8) >> 
    tag!("HTTP/") >> 
    version: map_res!(take_until!("\r"), str::from_utf8) >> 
    (Request { 
        method: method, 
        url: url.into(), 
        version: version.into()
        }
    ))));

    
fn run_parser(input: CompleteStr)
{
    match parse_request(input.as_bytes()) {
        IResult::Ok((rest, value)) => println!("Rest: {:?} Value : {:?}", rest, value),
        IResult::Err(err) => println!("Error: {:?}", err),
        
        
    }
}
fn main() {
    let get = CompleteStr::from("GET /home/ HTTP/ 1.1\r\n\0");
    run_parser(get);
    let post = CompleteStr::from("POST /update/ HTTP/1.1\r\n\0");
    run_parser(post);
    let wrong = CompleteStr::from("WRONG /wrong/ HTTP/1.1\r\n\0");
    run_parser(wrong);
}

```

```rust
#[macro_use]
extern crate nom;

use std::str;
use nom::{
    IResult,
    ErrorKind,
};

#[derive(Debug)]
enum Method{
    GET,
    POST,
}

#[derive(Debug)]
struct Request {
    method: Method,
    url: String,
    version: String,
}


named!(parse_method<&[u8], Method>, 
    return_error!(ErrorKind::Custom(12), alt!(map!(tag!("GET"), |_| 
    Method::GET) | map!(tag!("POST"), |_| Method::POST))));

named!(parse_request<&[u8], Request>, ws!(do_parse!(
    method: parse_method >>
    url: map_res!(take_until!(" "), str::from_utf8) >> 
    tag!("HTTP/") >> 
    version: map_res!(take_until!("\r"), str::from_utf8) >> 
    (Request { 
        method: method, 
        url: url.into(), 
        version: version.into()
        }
    ))));

    
fn run_parser(input: &str)
{
    match parse_request(input.as_bytes()) {
        IResult::Ok((rest, value)) => println!("Rest: {:?} Value : {:?}", rest, value),
        IResult::Err(err) => println!("Error: {:?}", err),
        
        
    }
}
fn main() {
    let get = "GET /home/ HTTP/ 1.1\r\n\0";
    run_parser(get);
    let post = "POST /update/ HTTP/1.1\r\n\0";
    run_parser(post);
    let wrong = "WRONG /wrong/ HTTP/1.1\r\n\0";
    run_parser(wrong);
}

```



```rust
let get = format!("{}\0", get);
```



## 최근 버전

```rust
extern crate nom;

use std::str;
use nom::{
    IResult,
    bytes::complete::{tag, take_until},
    combinator::map,
    branch::alt,
  };

#[derive(Debug)]
enum Method{
    GET,
    POST,
}

#[derive(Debug)]
struct Request {
    method: Method,
    url: String,
    version: String,
}


fn parse_method(input: &str) -> IResult<&str, Method>{
    let (input, method) = alt((map(tag("GET "), |_| Method::GET), map(tag("POST "), |_| Method::POST)))(input)?;
    Ok((input, method))
}

fn parse_url(input: &str) -> IResult<&str, String>{
    let (input, url) = take_until(" ")(input)?;
    Ok((input, String::from(url)))
}

fn parse_version(input: &str) -> IResult<&str, String>{
    let (input, _) = tag(" HTTP/")(input)?;
    let (input, version) = take_until("\r")(input)?;
    Ok((input, String::from(version)))
}

fn parse_request(input: &str) -> IResult<&str, Request>{
    let (input, method) = parse_method(input)?;
    let (input, url) = parse_url(input)?;
    let (_input, version) = parse_version(input)?;
    Ok(("",  Request {method:method, url:url, version:version}))
}

fn run_parser(input: &str)
{
    match parse_request(input) {
        IResult::Ok((rest, value)) => println!("Rest: {:?} Value : {:?}", rest, value),
        IResult::Err(err) => println!("Error: {:?}", err),
                
    }
}


fn main() {
    let get = "GET /home/ HTTP/ 1.1\r\n\0";
    run_parser(get);
    let post = "POST /update/ HTTP/1.1\r\n\0";
    run_parser(post);
    let wrong = "WRONG /wrong/ HTTP/1.1\r\n\0";
    run_parser(wrong);
}

```


