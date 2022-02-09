# Awaiting the future

아직 이 시기에는 futures가 외부 크레이트의 형태로 존재했다. 때문에 이걸 비동기 처리 해주기 위해서는 또다른 외부 크레이트인 `futures-await`를 사용해야했다. 하지만 이제는 futures가 스탠다드로 들어가고 동시에 비동기를 위한 문법으로 `async/await`가 도입됐기 때문에 따로 책에서 나오는 내용을 다룰 필요가 없다. 이미 이전 스터디에서도 여러 번 나온 얘기라 잘 기억이 나지 않는다면 3, 4주차 스터디 발표에서 나온 내용을 참고하면 될 것이다.

---

# 8. Asynchronous programming

> 비동기 프로그래밍에 대한 부분을 좀 더 자세히 살펴보기 위해 「Rust for Rustaceans」 책을 정리한다.

평소에 우리가 프로그래밍을 할 때에는 동기화 프로그래밍이 대부분이다. 그 이유는 다루기 쉽고 line by line으로 처리되기 때문에 복잡하지 않다. 하지만 이는 싱글 스레드 환경에서는 유용한 것이 분명하지만 현대의 멀티 스레드 환경에서는 매우 큰 낭비라는 것을 알 수 있다. 하나의 스레드에서 동기화된 작업을 진행하고 있다면 다른 스레드에서는 대기(blocked)하면서 아무것도 할 수 없는 상태로 존재하기 때문이다. 그래서 우리는 멀티스레딩(multithreading)을 이용해 병렬성(concurrency or parallel)을 가지는 비동기 프로그래밍을 할 것이다.

rust를 활용한 multithreading 구현은 챕터 10에서 더 자세히 다룬다고 한다. 여기서는 비동기를 위해 rust가 제공하고 있는 인터페이스들을 소개한다. 이 인터페이스들을 사용해 이후 `Mutex<T>`, `Arc<T>`와 같은 구조체를 활용해서 fearless concurrency를 구현한다.

### Asynchronous Interfaces

비동기 또는 논블로킹을 위한 인터페이스란 `Poll<T>` enum을 리턴하는 메소드를 말한다.

```rust
enum Poll {
	Ready(T),
	Pending
}
```

이 enum은 보통 poll로 시작하는 함수들의 리턴 타입으로 많이 사용된다. 이 함수들을 사용해서 비동기를 구현할 때 Poll::Ready를 반환할 때까지 loop를 돈다.

### Standardized Polling

그런데 여러 라이브러리에서 각자의 poll 함수를 작성해서 사용한다면 사용자 입장에서는 그 많은 스펙에 맞춰 각각 동작을 만들어줘야 할 것이다. 이런 일을 방지하기 위해서 polling을 표준화한 것이 바로 `Future` trait이다.

```rust
trait Future {
	type Output;
	fn poll(&mut self) -> Poll<Self::Output>;
}
```

아직 이용할 수는 없지만 Output에 지정된 타입이 나중에 생성될 예정이라는 것을 나타낸다. Future<Output = Foo> 같은 형태로 사용되는 것을 본 적이 있을 것이다. 이 trait을 이용해서 poll_recv 같은 함수를 각자 생성하지 않고 `impl Future`로 implementation을 통해 일관된 형태로 Future trait를 가지게 할 수 있다.

### Ergonomic Futures

```rust
async fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
	 while let Some(t) = rx.next().await {
		 tx.send(t).await;
	 }
}
```

여기 rx 채널로부터 메시지를 받아서 tx로 그대로 보내는 동작을 수행하는 비동기 함수 forward()가 있다. 매우 간단한 역할을 하는 함수지만 `async/await` 없이 `Future` trait만으로 이 동작을 구현하려고 한다면 어떻게 만들어야 할까?

```rust
enum Forward<T> {
    WaitingForReceive(ReceiveFuture<T>, Option<Sender<T>>),
    WaitingForSend(SendFuture<T>, Option<Receiver<T>>),
}
impl<T> Future for Forward<T> {
    type Output = ();
    fn poll(&mut self) -> Poll<Self::Output> {
        match self {
            Forward::WaitingForReceive(recv, tx) => {
                if let Poll::Ready((rx, v)) = recv.poll() {
                    if let Some(v) = v {
                        let tx = tx.take().unwrap();
                        *self = Forward::WaitingForSend(tx.send(v), Some(rx));
                        // Try to make progress on sending.
                        return self.poll();
                    } else {
                        // No more items.
                        Poll::Ready(())
                    }
                } else {
                    Poll::Pending
                }
            }
            Forward::WaitingForSend(send, rx) => {
                if let Poll::Ready(tx) = send.poll() {
                    let rx = rx.take().unwrap();
                    *self = Forward::WaitingForReceive(rx.receive(), Some(tx));
                    // Try to make progress on receiving.
                    return self.poll();
                } else {
                    Poll::Pending
                }
            }
        }
    }
}
```

위 코드가 바로 그 결과다. 책에서도 표현했듯이 그로테스크하다. 하나하나의 동작을 설명하기엔 길어서 코드만 보고 이해가 조금 부족할 것 같은 사람은 책을 참고해주길 바란다 (p.122~123). 적당히 요약하자면 poll 함수 내부에서 Self로 계속 점검하는데 메시지 왔는지 확인하고 안왔으면 Poll::Pending으로 유지, Poll::Ready 왔으면 정해진 동작 수행하고 자기 자신 또 불러서 다른 상태로 전환. 이런 식으로 receive, send를 왔다갔다 한다.

### async/await

```rust
async fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
	 while let Some(t) = rx.next().await {
		 tx.send(t).await;
	 }
}
```

이제 이 코드가 얼마나 간단하게 작성된 것인지 알 수 있다. 즉 Rust 1.39부터 등장한 `async/await` 덕분이다. 여기서 컴파일러가 사용하는 generator를 보도록 하자. 위 코드의 desugar 된 형태다.

```rust
generator fn forward<T>(rx: Receiver<T>, tx: Sender<T>) {
    loop {
        let mut f = rx.next();
        let r = if let Poll::Ready(r) = f.poll() {
            r
        } else {
            yield
        };
        if let Some(t) = r {
            let mut f = tx.send(t);
            let _ = if let Poll::Ready(r) = f.poll() {
                r
            } else {
                yield
            };
        } else {
            break Poll::Ready(());
        }
    }
}
```

처음 보는 syntax로 `yield`가 보인다. 운영체제에서 나오는 개념과 같이 yield가 불렸을 때 현재 상태(state)를 저장해놓고 다시 돌아오면 yield가 불렸던 시점부터 다시 전개되는 것이다. Ergonomic Futures 파트에서 나온 코드처럼 self를 통해 다시 poll()을 불렀을 때와 같은 동작이지만 훨씬 간단하다.

물론 위 코드에서 보이는 generator syntax는 rust 코드에서 사용할 수는 없고 컴파일러만이 async/await 구현할 때 위와 같은 generator를 통해 사용하고 있다.

직접 구현했던 Ergonomic Futures 파트의 코드와 async/await로 구현한 코드는 둘 다 같은 동작을 수행한다. 하지만 미묘하면서도 중요한 차이점이 존재하는데 그건 바로 async/await에선 &mut self를 사용할 수 있지만 직접 구현했을 때(Future trait로 했을 때)는 self만을 사용할 수 있다는 것이다. 그 이유는 수동으로 구현한 코드에선 borrow checker가 Receiver::next 동작 이후 해결하고 다시 돌아오는 코드 사이에 참조될 수 없어서 컴파일 거부하고, async/await 코드에서는 컴파일러가 state machine(아마 generator 말하는듯?)로 전환되기 전에 borrow checker가 참조에 대해 검사할 수 있고 rx가 future의 드롭 이후에도 접근 불가능한걸 확인할 수 있기 때문이다.
