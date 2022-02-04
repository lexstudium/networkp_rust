# w5 내용 정리

# Working with streams and sinks

- Future가 Result에 대응되면, Stream은 Iterator에 대응된다.
- 이번에는 책 예시코드가 그대로 돌아가서 책 코드도 보고 새로워진 부분은 뽑아서 볼 것

```toml
[package]
name = "streams"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
futures = "0.1.17"
rand = "0.3.18"

```



## stream

```rust
trait Stream {
    type Item;
    type Error;
    fn poll(& mut self) -> Poll<Option<Self::Item>, Self::Error>;
}
```

- recap Iterator

- ```rust
  impl Iterator for {내 struct}{
      type Item = {item type};
      
      fn next(&mut self) -> Option<Self::Item>{
          // do sth
          if not done{
              Some(val)
          } else {
              None
          }
      }
  }
  ```

- ```rust
  impl Stream for {내 struct}{
      type Item = {item type};
      // Error 가 필요해짐
      type Error = {Error type};
      fn poll(&mut self) -> Poll<Option<Self::Item>, io::Error>{
          // do something
  
          if not done {
              Ok(Async::Ready(Some(self.current)))            
          } else {
              Ok(Async::Ready(None))
          }
      }
  }
  ```

- 



#### collatz stream

```rust
use std::{io, thread};
use std::time::Duration;
use futures::stream::Stream;
use futures::{Poll, Async};
use rand::{thread_rng, Rng};
use futures::Future;

#[derive(Debug)]
struct CollatzStream{
    current : u64,
    end : u64,
}

impl CollatzStream {
    fn new(start: u64) -> CollatzStream {
        CollatzStream{
            current : start,
            end : 1,
        }
    }
}

impl Stream for CollatzStream{
    type Item = u64;
    type Error = io::Error;
    fn poll(&mut self) -> Poll<Option<Self::Item>, io::Error>{
        let d = thread_rng().gen_range::<u64>(1,5);
        thread::sleep(Duration::from_secs(d));
        if self.current % 2 == 0 {
            self.current = self.current / 2;

        } else {
            self.current = self.current*3 + 1;
        }

        if self.current == self.end {
            Ok(Async::Ready(None))
        } else {
            Ok(Async::Ready(Some(self.current)))
        }
    }
}
```



```rust


fn main() {
    let stream = CollatzStream::new(10);
    let f = stream.for_each(|num| {
        println!("{}", num);
        Ok(())
    });

    f.wait().ok();
}

```

- f가 future를 리턴해서 wait하고 ok해서 block하고 모든 결과를 가져온다.



## ping pong

- future가 통신하기 위한 channel

```rust
use std::thread;
use std::fmt::Debug;
use std::time::Duration;
use futures::Future;
use rand::{thread_rng, Rng};

use futures::sync::mpsc;
use futures::{Sink, Stream};
use futures::sync::mpsc::Receiver;

fn sender() -> &'static str {
    let mut d = thread_rng();
    thread::sleep(Duration::from_secs(d.gen_range::<u64>(1,5)));
    d.choose(&["ping", "pong"]).unwrap()
}

fn receiver<T: Debug>(recv: Receiver<T>){
    let f = recv.for_each(|item| {
        println!("{:?}", item);
        Ok(())
    });
    f.wait().ok();
}


fn main() {
    let (tx, rx) = mpsc::channel(100);
    let h1 = thread::spawn(|| {
        tx.send(sender()).wait().ok();
    });
    let h2 = thread::spawn(|| {
        receiver::<&str>(rx);
    });

    h1.join().unwrap();
    h2.join().unwrap();
}

```

- mpsc::channel(100) : buffer size

## BiLock

- sync에서 std sync Mutex를 mirro한 것

```rust
use std::thread;
use std::fmt::Debug;
use std::time::Duration;
use futures::{Future, Async};
use rand::{thread_rng, Rng};

use futures::sync::{mpsc, BiLock};
use futures::{Sink, Stream};
use futures::sync::mpsc::Receiver;


fn sender(send: &BiLock<u64>) -> &'static str {
    match send.poll_lock() {
        Async::Ready(mut lock) => *lock += 1,
        Async::NotReady => ()
    }
    let mut d = thread_rng();
    thread::sleep(Duration::from_secs(d.gen_range::<u64>(1,5)));
    d.choose(&["ping", "pong"]).unwrap()
}

fn receiver<T: Debug>(recv: Receiver<T>, recv_lock: BiLock<u64>){
    match recv_lock.poll_lock() {
        Async::Ready(lock) => println!("Value of lock {}", *lock),
        Async::NotReady => ()
    }
    let f = recv.for_each(|item| {
        println!("{:?}", item);
        Ok(())
    });
    f.wait().ok();
}

fn main() {
    let counter = 0;
    let (send, recv) = BiLock::new(counter);
    let (tx, rx) = mpsc::channel(100);
    let h1 = thread::spawn(move || {
        tx.send(sender(&send)).wait().ok();
    });
    let h2 = thread::spawn(|| {
        receiver::<&str>(rx, recv);
    });

    h1.join().unwrap();
    h2.join().unwrap();

}

```

- BiLock의 constructor는 2가지 handle을 준다.



## 최근 버전

### stream

- streamExt를 가져와야 사용 가능하다.
- streamExt에 iteration 관련된 것들이 있고, 이건 자동으로 구현됨

```rust
use std::thread;
use std::time::Duration;
use futures::stream::{Stream, StreamExt};
use futures::task::{Poll, Context};
use futures::executor;
use rand::{thread_rng, Rng};
use core::pin::Pin;


#[derive(Debug)]
struct CollatzStream{
    current : u64,
    end : u64,
}

impl CollatzStream {
    fn new(start: u64) -> CollatzStream {
        CollatzStream{
            current : start,
            end : 1,
        }
    }
}

impl Stream for CollatzStream{
    type Item = u64;
    fn poll_next(mut self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<Option<Self::Item>>{
        let d = thread_rng().gen_range(1..3);
        thread::sleep(Duration::from_secs(d));
        if self.current % 2 == 0 {
            self.current = self.current / 2;

        } else {
            self.current = self.current*3 + 1;
        }

        if self.current == self.end {
            Poll::Ready(None)
        } else {
            Poll::Ready(Some(self.current))
        }
    }
}

fn main() {
    let mut _stream = CollatzStream::new(10);

    let fut_values = async {

        let mut pending = vec![];
        while let Some(v) = _stream.next().await {
            pending.push(v);
        }

        pending
    };

    let values: Vec<u64> = executor::block_on(fut_values);

    println!("Values={:?}", values);
}

// Values=[5, 16, 8, 4, 2]
```



### 그냥 숫자 전달

- ```toml
  [package]
  name = "future-ping-pong"
  version = "0.1.0"
  edition = "2021"
  
  # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
  
  [dependencies]
  futures = {version = "0.3.19", features = ["thread-pool"]}
  tokio-core = "0.1.18"
  rand = "0.8.4"
  ```

- Cargo.toml에 thread-pool을 추가해야한다. 그래야 ThreadPool 사용 가능



### 문법 정리

#### channel

```rust
let pool = ThreadPool::new().expect("Failed to build pool");
let (tx, mut rx) = mpsc::unbounded::<i32>();
```

- mpsc와 oneshot이 있다.

- mpsc : mutilple producer single consumer

- oneshot : single producer single consumer

  ```rust
  use futures::channel::oneshot;
  use std::{thread, time::Duration};
  
  let (sender, receiver) = oneshot::channel::<i32>();
  
  thread::spawn(|| {
      println!("THREAD: sleeping zzz...");
      thread::sleep(Duration::from_millis(1000));
      println!("THREAD: i'm awake! sending.");
      sender.send(3).unwrap();
  });
  
  println!("MAIN: doing some useful stuff");
  
  futures::executor::block_on(async {
      println!("MAIN: waiting for msg...");
      println!("MAIN: got: {:?}", receiver.await)
  });
  ```

#### unbounded vs channel

- unbounded : 메모리가 허가하는 한 최대한

- channel은 buffer size 존재

- ```rust
  let (tx, mut rx) = mpsc::unbounded::<i32>();
  let (tx, rx) = mpsc::channel(100);
  ```

- 어떤 데이터를 주고 받을 것인지 정해야한다. 

- block_on : future를 current thread에서 기다리는 것

  ```rust
  pub fn block_on<F>(f: F) -> <F as Future>::Output 
  where
      F: Future, 
  
  // Run a future to completion on the current thread.
  
  // This function will block the caller until the given future has completed.
  
  // Use a LocalPool if you need finer-grained control over spawned tasks.
  ```

  

```rust
use futures::channel::mpsc;
use futures::executor::ThreadPool;
use futures::executor; 
use futures::StreamExt;

fn main() {
    let pool = ThreadPool::new().expect("Failed to build pool");
    let (tx, mut rx) = mpsc::unbounded::<i32>();
    let fut_values = async {
        
        let fut_tx_result = async move {
            (0..20).for_each(|v| {
                tx.unbounded_send(v).expect("Failed to send");
                println!("send : {}", v);
            })
        };

        pool.spawn_ok(fut_tx_result);

        let mut pending = vec![];
        
        while let Some(v) = rx.next().await {
            pending.push(v * 2);
            println!("received {}", v);
        }

        pending
    };

    let values: Vec<i32> = executor::block_on(fut_values);

    println!("Values={:?}", values);
}


```

- 

```rust
send : 0
send : 1
send : 2
send : 3
send : 4
send : 5
send : 6
send : 7
send : 8
send : 9
send : 10
send : 11
received 0
received 1
send : 12
send : 13
send : 14
send : 15
send : 16
send : 17
send : 18
send : 19
received 2
received 3
received 4
received 5
received 6
received 7
received 8
received 9
received 10
received 11
received 12
received 13
received 14
received 15
received 16
received 17
received 18
received 19
Values=[0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36, 38]
```

