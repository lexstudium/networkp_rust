# Pin and Unpin

## Pin

self-referential한 type을 만들 때 필요합니다.

ex)

```rust
struct A {
    x: i32,
    b: &i32
}

x = 1;
b = &x;
```

저 상태에서 `A`를 통째로 옮겨버리면 `b`의 값이 invalid해집니다. 그럼 unsafe하죠? 그래서 `A`라는 struct에 통째로 Pin을 감쌉니다. 그럼 `A`를 움직일 수 없죠.

```rust
struct Pin<P> { pointer: P }

impl<P> Pin<P> where P: Deref {
    pub unsafe fn new_unchecked(pointer: P) -> Self;
}

impl<'a, T> Pin<&'a mut T> {
    pub unsafe fn get_unchecked_mut(self) -> &'a mut T;
}

impl<P> Deref for Pin<P> where P: Deref {
    type Target = P::Target;
    fn deref(&self) -> &Self::Target;
}
```

`P`가 `Deref`를 구현해야합니다. 즉, `Box`, `Rc` 등등의 포인터만 `P`로 사용가능합니다. 위의 예시와 연결지어서 설명하면 `Pin<Box<A>>` 이런 식으로요. 여기서 포인터 `P`의 타입은 `Box<A>`가 되겠네요.

### `new_unchecked`

constructor가 unsafe합니다. 왜냐면 `P`의 타입만 보고는 걔가 self-referential한 data를 만들지 아닐지 알 수가 없기 때문입니다.

### `get_unchecked_mut`

`P`를 mutable하게 꺼내는 메소드입니다. 얘도 unsafe하죠. 이것도 당연한 게, 여기서 `P`를 꺼낸 다음에 다른 `P`랑 `mem::swap`으로 바꿔치기 해버리면 Pin이 깨집니다.

### `deref`

놀랍게도 얘는 unsafe하지 않습니다. mutable하지 않게 포인터를 꺼내면, 값을 바꿀 수가 없기 때문에 Pin이 깨지지 않기 때문입니다.

## Unpin

Pin과 반대인 포인터들, 즉 옮겨도 상관없는 포인터들은 전부 Unpin. 대부분의 primitive는 Unpin을 자동으로 구현합니다.

# Waker

future를 만들었으면, 걔가 언제 사용가능한지 확인해야겠죠? 그걸 확인하는 친구가 *executor*입니다. `future.poll()`을 해서 `Poll::Pending`이면 기다렸다가 나중에 다시 poll을 합니다. `Poll::Ready`가 나올 때까지. **'나중에 다시'**라는 말이 애매하죠?

아주 오래 걸리는 io를 future로 묶어서 asynchronous하게 처리했다고 치겠습니다. 루프를 돌면서 `future.poll()`을 계속 하나요? 그건 너무 비효율적이죠?

그래서 

```rust
trait Future {

type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

저기 보면 `Context`가 있죠? 저 안에 `Waker`라는 친구가 있습니다. future가 progress가 있다 싶으면 `Waker`가 future를 깨웁니다. 그때 poll을 호출하면 됩니다.

future는 크게 2 종류로 나눌 수 있습니다.
- 내부에 다른 future가 있는 경우
- 내부에 다른 future가 없는 경우

전자는 poll하기 되게 쉬워요. 내부의 child future 중에서 `Poll::Pending`인 친구가 하나라도 있으면 parent도 `Poll::Pending`입니다. 생각해보면 당연합니다. inner future들이 전부 처리돼야 parent도 처리될 테니까요.

내부에 다른 future가 없으면? 얘의 `Waker`는 외부에 있겠죠? 여기서 또 2가지 경우의 수가 있습니다.
- 같은 프로세스 안에 `Waker`가 있는 경우
  - ex) channel receiver
- 다른 프로세스에 `Waker`가 있는 경우

전자는 간단합니다. 위에서 예시처럼 channel receiver에서 메시지를 받으면 future가 걔를 처리한다고 치겠습니다. 그럼 메시지를 받는 코드에 뒤에 future의 `Waker`를 호출하는 코드를 명시적으로 추가해주면 됩니다.

근데 후자의 경우는요? 후자는 대부분 OS에서 만드는 이벤트와 관련돼 있습니다. OS의 이벤트와 관련된 코드 안에 future와 관련된 코드를 넣는 것은 불가능하니까, executor가 OS와 상호작용하면서 적절한 때에 `wake`함수를 호출합니다. OS에서 만든 모든 이벤트를 다 확인해보고 깨울 future가 있는지 확인합니다. 깨울 future가 없으면 executor는 잠들어요. OS에서 새로운 이벤트가 발생하면 다시 executor가 일어나서 깨울 future를 확인하죠.

이렇게 하면 서로 다른 executor끼리 호환이 안된다는 단점이 있다고 하네요. 여러 executor를 아울러서 generic하게 만들고 싶으면 `T: AsyncRead + AsyncWrite` 같은 방식으로 하면 됩니다. 근데 아직 Rust 전체를 아우르는 de facto standard가 없다고 하네요.

그 다음 부분에는 '`Waker`가 사실은 executor를 깨우는 게 아니고 future를 깨우는 거다'라고 말하고 있는데 자세하게는 잘 이해가 안 가네요...

그 다음 부분도 이해가 조금 안 가요. 그냥 `async`, `await`만 사용하면 한번에 하나의 future만 기다리면 되지만, `select`나 `join`등의 메소드를 사용하면 한번에 여러개의 future를 기다려야합니다. 한번에 여러개를 기다리면, 누구를 언제 `poll`할지가 애매하죠. 매번 모든 future에 `poll`을 하기는 비효율적이고, 그래서 subexecutor를 사용해서 누구를 언제 poll할지 결정한다네요... 자세히는 이해가 안 갑니다 ㅠㅠ

# Spawning

아주 간단한 서버와 클라이언트를 만들어보겠습니다.

```rust
async fn handle_client(socket: TcpStream) -> Result<()> {
    // 클라이언트가 하고 싶은 거 마음껏 처리
}

async fn server(socket: TcpListener) -> Result<()> {

    while let Some(stream) = socket.accept().await? {
        handle_client(stream).await?;
    }

}
```

클라이언트가 뭘하는지는 전혀 중요하지 않고, 서버의 구현만 보겠습니다. 위의 방식대로 구현하는 것은 전혀 asynchoronous하지 않습니다. 새로운 요청이 들어올 때까지 계속 대기하고, 요청이 하나 들어오면 그 요청의 처리가 끝날 때까지 계속 대기합니다.

```rust
async fn server(socket: TcpListener) -> Result<()> {
    let mut clients = Vec::new();

    loop {
        poll_client_futures(&mut clients)?;

        if let Some(stream) = socket.try_accept()? {
            clients.push(handle_client(stream));
        }

    }

}
```

아까보다는 조금 더 효율적입니다. 클라이언트들이 모여 있는 벡터를 하나 만들었습니다. 요청이 들어올 때마다 그 요청을 클라이언트 묶음에 넘기고, 클라이언트 묶음은 요청들을 병렬적으로 처리합니다. 그래도 조금은 아쉬운 점이 있습니다. 매 루프마다 모든 클라이언트를 확인하는 게 조금 비효율적입니다. 또한 이렇게하면 멀티 스레딩이 되지 않습니다. poll_client_futures에 `&mut clients`를 넘기는게 저 친구가 스레드를 넘나들 수 없어서 그렇다는 거 같네요.

```rust
async fn server(socket: TcpListener) -> Result<()> {
    while let Some(stream) = socket.accept().await? {
        // Spawn a new task with the Future that represents this client.
        // The current task will continue to just poll for more connections
        // and will run concurrently (and possibly in parallel) with handle_client.
        spawn(handle_client(stream));
    }
}
```

마지막 방법입니다. 요청이 들어오면 아예 새로운 task를 spawn해서 거기로 요청을 넘겨버립니다.

저기서 spawn이 뭐하는 함수인지 알아보려 했는데 executor의 구현에 따라 spawn의 구현도 다양하다고 하네요. 대부분은 새로운 스레드를 열어서 거기서 요청을 처리하고 알아서 poll을 하는 방식인 거 같습니다.
