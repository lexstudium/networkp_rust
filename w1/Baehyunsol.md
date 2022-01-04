# Generics and Traits

## Generics


```rust
struct Tuple<T> {  // 동일한 type 2개를 원소로 갖는 Tuple
    first: T,
    second: T,
}

fn main() {
    let tuple_u32: Tuple<u32> = Tuple {first: 4u32, second: 2u32 };  // `4u32`는 `4 as u32`와 동일한 의미
    let tuple_u64: Tuple<u64> = Tuple {first: 5u64, second: 6u64 };

    println!("{}", sum(tuple_u32));
    println!("{}", sum(tuple_u64));

    let tuple: Tuple<String> = Tuple {
        first: "One".to_owned(),
        second: "Two".to_owned() 
    };

    println!("{}", sum(tuple));
}

fn sum<T>(tuple: Tuple<T>) -> T {  // 컴파일 되지 않습니다. 나중에 다시 설명하겠습니다.
    tuple.first + tuple.second
}
```

C++에 익숙하신 분들은 저 코드가 무슨 코드인지 대략 짐작이 가실 겁니다.

`u32`, `u64`, `String` 등의 type으로 이뤄진 tuple들을 정의하고, `sum` 함수를 호출하고 있습니다.

Rust에서 Generic의 구현 방식은 C++과 비슷합니다. 새로운 type으로 `Tuple`이 정의될 때마다 그 type에 대한 `Tuple`의 정의를 새로 만들어서 코드에 포함시킵니다. 예를 들어서 `Tuple<String>`을 만들어서 `sum`을 호출하는 코드를 보면

```rust
struct TupleString {
    first: String,
    second: String
}

fn sum_string(tuple: TupleString) -> String {
    tuple.first + tuple.second
}
```

를 만드는 식으로요.

## Traits

```rust
struct Tuple<T> {  // 동일한 type 2개를 원소로 갖는 Tuple
    first: T,
    second: T,
}

fn main() {
    let tuple_u32: Tuple<u32> = Tuple {first: 4u32, second: 2u32 };  // `4u32`는 `4 as u32`와 동일한 의미
    let tuple_u64: Tuple<u64> = Tuple {first: 5u64, second: 6u64 };

    println!("{}", sum(tuple_u32));
    println!("{}", sum(tuple_u64));

    let tuple: Tuple<String> = Tuple {
        first: "One".to_owned(),
        second: "Two".to_owned() 
    };

    println!("{}", sum(tuple));
}

fn sum<T>(tuple: Tuple<T>) -> T {
    tuple.first + tuple.second
}
```

이 예시를 계속 가지고 설명하겠습니다. 만약 덧셈이 정의되지 않은 type `T`를 이용해서 `Tuple`을 만든 뒤 `sum` 함수를 호출하면 어떻게 될까요? C++은 개의치 않고 generic을 생성합니다. 그리고 한참 나중에 `T`에 대한 덧셈은 시도하다가 그때서야 컴파일 에러를 던집니다. 컴파일 에러만 보고는 무슨 일인지 파악하기 힘들죠. 그에 반해 Rust 컴파일러는 저런 상황을 미연에 막습니다. `sum` 함수 내부에서 `T` type 들끼리 덧셈을 하는 걸 파악하고는, `T` type이 덧셈이 가능한 type임을 명시하도록 강제합니다. 무슨 뜻인지 아래 코드를 이용해서 살펴보겠습니다.

```rust
fn sum<T: Add<Output = T>>(tuple: Tuple<T>) -> T {
    tuple.first + tuple.second
}
```

Generic 정의에서 `T` 뒤에 `Add<Output = T>`라는 문구가 추가되었죠? `T`라는 type은 `Add<Output = T>`라는 trait을 가져야한다는 뜻입니다.

Trait는 Rust 언어에서 아주 아주 중요한 개념 중 하나입니다. 다른 언어에서 상속, 인터페이스 등을 이용해서 구현하던 개념들은 Rust에선 전부 trait를 이용해서 합니다. 다른 예시를 보겠습니다. 아래 코드는 Rust로 만든 게임의 일부입니다.

```rust
pub fn get_visible_objects<T: Distance + IntoObject>(objects: &Vec<T>, pos: &Point, sight: f32) -> Vec<Rc<Object>> {
    objects.iter().filter(
        |obj|
        obj.get_distance(pos) < sight
    ).map(
        |obj|
        Rc::new(obj.into_object())
    ).collect()
}
```

함수의 내부는 이해하지 못하셔도 상관없습니다. `<T: Distance + IntoObject>` 이 부분만 보겠습니다. 만약 generic이나 trait가 없었다면 위의 함수는 게임 내의 모든 물체들에 대해서 각각 따로 정의해야 합니다. 아래와 같이 말이죠.

```rust
pub fn get_visible_zombie_objects(zombies: &Vec<Zombie>, pos: &Point, sight: f32) -> Vec<Rc<Object>> {
    zombies.iter().filter(
        |zombie|
        zombie.get_distance(pos) < sight
    ).map(
        |zombie|
        Rc::new(zombie.into_object())
    ).collect()
}


pub fn get_visible_item_objects(items: &Vec<Item>, pos: &Point, sight: f32) -> Vec<Rc<Object>> {
    items.iter().filter(
        |item|
        item.get_distance(pos) < sight
    ).map(
        |item|
        Rc::new(item.into_object())
    ).collect()
}

// ...
```

잘 보시면 함수의 내부가 동일합니다. 단지 이름만 다르죠. `get_distance`라는 메소드와 `into_object`라는 메소드만 정의되어 있으면 무슨 타입이든지 선언할 수 있는 함수입니다. 그래서 `Distance`라는 trait와 `IntoObject`라는 trait를 정의하고 게임 내의 오브젝트들에 저 trait들을 정의해주면 처음에 봤던 것처럼 generic으로 깔끔하게 함수를 정의할 수 있습니다.

```rust
pub trait Distance {
    fn get_distance(&self, from: &Point) -> f32;
}

pub trait IntoObject {
    fn into_object(&self) -> Object;
}

impl Distance for Zombie {
    fn get_distance(&self, from: &Point) -> f32 {
        // 생략
    }
}

impl IntoObject for Zombie {
    fn into_object(&self) -> Object {
        // 생략
    }
}

// Item, Player등 다른 type들도 동일한 방식으로 정의

pub fn get_visible_objects<T: Distance + IntoObject>(objects: &Vec<T>, pos: &Point, sight: f32) -> Vec<Rc<Object>> {
    objects.iter().filter(
        |obj|
        obj.get_distance(pos) < sight
    ).map(
        |obj|
        Rc::new(obj.into_object())
    ).collect()
}
```

책에 있는 예시도 살펴보겠습니다.

```rust
// trait 정의
trait Sawtooth {
    fn sawtooth(&self) -> Self;
}

// built-in type인 f64에도 Sawtooth를 정의할 수 있습니다.
impl Sawtooth for f64 {
    fn sawtooth(&self) -> f64 {
        self - self.floor()
    }
}

fn main() {
    println!("{}", 2.34f64.sawtooth());
}
```

### Derive

자주 쓰는 trait들은 더 간편한 macro로 정의되어 있습니다.

```rust
struct Point {
    x: f32,
    y: f32
}

fn main() {
    println!("{:?}", vec![1, 2, 3, 4, 5, 6]);  // 컴파일 됨
    println!("{:?}", Point {x: 1.0, y: 1.0});  // 컴파일 안됨
}
```

`Point`를 `println!`으로 출력하기 위해선 인스턴스를 문자열로 바꾸는 trait가 정의되어 있어야 합니다. standard library의 `std::fmt::Debug` trait를 구현하면 되는데요,

```rust
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

딱봐도 무시무시하게 생겼죠? 근데 생각해보면 어떤 struct의 내용물을 출력하는 방법은 특별할 게 없습니다. 그 struct의 이름을 출력하고, 멤버들의 이름과 값을 쭉 출력하면 그만입니다. 그래서 저것보다 훨씬 간편한 매크로가 기본적으로 정의돼 있습니다.

```rust
#[derive(Debug)]
struct Point {
    x: f32,
    y: f32
}
```

위와 같은 방법으로 `std::fmt::Debug` macro를 손쉽게 정의할 수 있습니다.

# Error Handling

Rust의 에러 처리 방식은 C++/Python 등과 조금 다릅니다. C++/Python에선 에러를 *던집*니다. 밖의 함수에서 에러를 받아서 처리해주면 다행이고, 그렇지 않으면 프로그램이 죽어버리거나 예상치 못한 곳에서 에러를 받을 수도 있습니다. Rust에선 `Result`라는 enum을 이용합니다. 즉, 에러가 날 가능성이 있는 함수들은 type 자체가 `Result`로 돼 있어서 에러처리를 무조건 하도록 강제합니다.

아래의 예시를 보겠습니다. `open`이라는 함수는 파일의 주소를 받아서 그 파일을 열고 return합니다.

```rust
fn open_file(path: &str) -> Result<File, Error>
```

C++/Python 같은 경우는 저 함수의 return 값이 `File`이고, 중간에 에러가 날 경우 에러를 *던집*니다. `open`을 호출하면서 에러 처리하는 코드를 같이 넣어두면 다행이지만 까먹을 경우 프로그램이 통째로 죽을 수도 있습니다. 하지만 Rust의 경우 return type 자체가 `Result<File, Error>`이기 때문에 에러 처리하는 코드를 넣지 않으면 컴파일 에러가 납니다. 왜냐면 `File`과 `Result<File>`은 서로 다른 type이거든요.

즉, `open_file` 함수를 호출하면 결과물은 `Ok(File)` 혹은 `Err(Error)`의 형태로 반환됩니다. 각각의 상황에 대해서 모두 처리하는 코드를 짜야합니다.

에러 처리하는 방법들을 살펴보겠습니다.

```rust
fn open_file_1(path: &str) -> File {
    open_file(path).unwrap()
}

fn open_file_2(path: &str) -> File {
    open_file(path).expect("Failed to open a file!")
}

fn open_file_3(path: &str) -> File {
    match open_file(path) {
        Ok(f) => f,
        Err(e) => {
            // 에러 처리하는 코드
        }
    }
}
```

`unwrap` 메소드와 `expect` 메소드는 비슷합니다. `Result`가 `Ok(File)`일 경우 그 안의 `File`을 그대로 return하고, `Err(Error)`일 경우 `panic!`을 하면서 프로그램을 통째로 종료시켜버립니다. 다만 `unwrap`은 기본 에러 메시지를 출력하고, `expect`는 프로그래머가 원하는 에러 메시지를 출력할 수 있습니다. 아니면 3번 함수에서 보듯 패턴 매칭을 이용해서 에러 처리하는 로직을 자세하게 짤 수도 있습니다.
