# Custom serialization and deserialization

Serde는 거의 모든 타입에게 built-in serialization, deserialization을 제공한다. 하지만 가끔 auto-implement가 실패할 때도 있다. 이럴 때를 위해 수동으로 구현할 수 있게 만들어졌다.

그렇지만 실제로 거의 사용되지 않을 확률이 높다. 그냥 이렇게 구현되나보다 하고 지나가면 될 것 같다.

## Custom serialization

* Environment Setting

> $ cargo new --bin serde-custom

```toml
# Cargo.toml
[package]
name = "serde-custom"
version = "0.1.0"
edition = "2021"
authors = ["Hyunmin Shin <shm1193@gmail.com>"]

[dependencies]
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
serde_test = "1.0"
```


* `Serialize` trait

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}
```
Generic Type은 특별할게 없다.

이 trait이 지원하는 `Serializer` methods 중, data structure에 맞춰 doc에 나온 규칙대로 작성해주면 된다.
책에 나오는 struct를 가지고 하기에는 작성이 간단하고 똑같으므로 지금 보여줄 예시는 다른 data type인 `struct_variant` 자료구조를 이용해 `Serializer`를 doc을 보며 만들어봤다. struct_variant의 형태로는 enum 내부에 struct가 존재하는 것으로 했다.

* `Serialize` Implementation

```rust
// serde::ser::SerializeStructVariant
#[derive(Debug, PartialEq)]
enum Ex1 {
    St1 { hello: i32, rust: i32, world: u8, karatus: f32 },
}

impl Serialize for Ex1 {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
            S: Serializer,
    {
        match *self {
            Ex1::St1 {
                ref hello,
                ref rust,
                ref world,
                ref karatus,
            } => {
                let mut sv = serializer.serialize_struct_variant("Ex1", 0, "St1", 4)?;
                sv.serialize_field("hello", hello)?;
                sv.serialize_field("rust", rust)?;
                sv.serialize_field("world", world)?;
                sv.serialize_field("karatus", karatus)?;
                sv.end()
            }
        }
    }
}
```
struct variant 내부에 각자 다른 data type을 부여해도 정상적으로 동작하는 것을 확인할 수 있다.
비교적 쉽게 구현 가능하다.

## Custom deserialization

구현이 상대적으로 쉬운 Serialize와는 달리 Deserialize의 구현은 복잡하다.

* `Deserialize` trait
 
```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```
`Serialize` trait의 Generic Type과는 달리 `Deserialize` trait의 Generic Type은 추가로 lifetime을 나타내는 `'de`가 존재한다. serialized data를 가져오고 deserialization 할 때 reference 같은 데이터 타입들을 zero-copy 해서 efficiency를 높이기 위해서다.

그리고 `Deserialize` trait에서는 `Deserializer` trait과 `Visitor` trait을 통해 실구현한다. 두 traits 사이의 관계는 다음과 같다.

> Deserializer - input data를 다양한 data types으로 mapping
> Visitor		    - input data의 타입에 따라 실제 처리 부분의 구현 (Deserializer 함수 내부에서 호출됨)

* `Visitor` trait

```rust
/// A visitor that deserializes a long string - a string containing at least
/// some minimum number of bytes.
struct LongString {
    min: usize,
}

impl<'de> Visitor<'de> for LongString {
    type Value = String;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        write!(formatter, "a string containing at least {} bytes", self.min)
    }

    fn visit_str<E>(self, s: &str) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        if s.len() >= self.min {
            Ok(s.to_owned())
        } else {
            Err(de::Error::invalid_value(Unexpected::Str(s), &self))
        }
    }
}
```
위의 `Visitor` 예시는 상황에 맞춰 visit_str() 함수만을 구현했지만 Serde에서 지원하는 다른 data types도 있으니 doc을 참고하면 된다.

* `Deserialize` Implementation

책에서는 다른 말 없이 그냥 이렇게 이렇게 만들면 된다고 했지만 아무것도 모르는 상태로 따라만 하기엔 좀 그래서 좀 더 찾아봤다. 그러다 doc에 [가이드](https://serde.rs/deserialize-struct.html) 부분이 있어 같은 포멧으로 작성했다는 것을 알았다. 그래서 다른 예시를 들고 싶었지만 너무 단순한 구현이라 구조를 분석하는 방향으로 틀었다.

**Step 1. `enum Field` 선언**
```rust
#[derive(Debug, PartialEq)]
struct KubeConfig {
    port: u8,
    healthz_port: u8,
    max_pods: u8,
}

impl<'de> Deserialize<'de> for KubeConfig {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        enum Field {
            Port,
            HealthzPort,
            MaxPods,
        }

		impl<'de> Deserialize<'de> for Field {
            fn deserialize<D>(deserializer: D) -> Result<Field, D::Error>
            where
                D: Deserializer<'de>,
            {
                struct FieldVisitor;

                impl<'de> Visitor<'de> for FieldVisitor {
                    type Value = Field;

                    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                        formatter.write_str("`port` or `healthz_port` or `max_pods`")
                    }

                    fn visit_str<E>(self, value: &str) -> Result<Field, E>
                    where
                        E: de::Error,
                    {
                        match value {
                            "port" => Ok(Field::Port),
                            "healthz_port" => Ok(Field::HealthzPort),
                            "max_pods" => Ok(Field::MaxPods),
                            _ => Err(de::Error::unknown_field(value, FIELDS)),
                        }
                    }
                }

                deserializer.deserialize_identifier(FieldVisitor)
            }
        }
    }
```
`KubeConfig` struct의 field name을 String으로 할당해서 가지고 있는 걸 피하기 위해 &str을 사용하는 `Field` enum을 사용한다.

**Step 2. `struct  KubeConfigVisitor` 선언 및 `Visitor` trait 구현**
```rust
impl<'de> Deserialize<'de> for KubeConfig {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        struct KubeConfigVisitor;

        impl<'de> Visitor<'de> for KubeConfigVisitor {
            type Value = KubeConfig;

            fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                formatter.write_str("struct KubeConfig")
            }

            fn visit_map<V>(self, mut map: V) -> Result<KubeConfig, V::Error>
            where
                V: MapAccess<'de>,
            {
                let mut port = None;
                let mut hport = None;
                let mut max = None;
                while let Some(key) = map.next_key()? {
                    match key {
                        Field::Port => {
                            if port.is_some() {
                                return Err(de::Error::duplicate_field("port"));
                            }
                            port = Some(map.next_value()?);
                        }
                        Field::HealthzPort => {
                            if hport.is_some() {
                                return Err(de::Error::duplicate_field("healthz_port"));
                            }
                            hport = Some(map.next_value()?);
                        }
                        Field::MaxPods => {
                            if max.is_some() {
                                return Err(de::Error::duplicate_field("max_pods"));
                            }
                            max = Some(map.next_value()?);
                        }
                    }
                }
                let port = port.ok_or_else(|| de::Error::missing_field("port"))?;
                let hport = hport.ok_or_else(
                    || de::Error::missing_field("healthz_port"),
                )?;
                let max = max.ok_or_else(|| de::Error::missing_field("max_pods"))?;
                Ok(KubeConfig {
                    port: port,
                    healthz_port: hport,
                    max_pods: max,
                })
            }
        }

        const FIELDS: &'static [&'static str] = &["port", "healthz_port", "max_pods"];
        deserializer.deserialize_struct("KubeConfig", FIELDS, KubeConfigVisitor)
    }
}
```


```rust
// https://github.com/serde-rs/json/blob/master/src/de.rs#L1804
// deserialize.deserialize_struct()

fn deserialize_struct<V>(
    self,
    _name: &'static str,
    _fields: &'static [&'static str],
    visitor: V,
) -> Result<V::Value>
where
    V: de::Visitor<'de>,
{
    let peek = match tri!(self.parse_whitespace()) {
        Some(b) => b,
        None => {
            return Err(self.peek_error(ErrorCode::EofWhileParsingValue));
        }
    };

    let value = match peek {
        b'[' => {
            check_recursion! {
                self.eat_char();
                let ret = visitor.visit_seq(SeqAccess::new(self));
            }

            match (ret, self.end_seq()) {
                (Ok(ret), Ok(())) => Ok(ret),
                (Err(err), _) | (_, Err(err)) => Err(err),
            }
        }
        b'{' => {
            check_recursion! {
                self.eat_char();
                let ret = visitor.visit_map(MapAccess::new(self));
            }

            match (ret, self.end_map()) {
                (Ok(ret), Ok(())) => Ok(ret),
                (Err(err), _) | (_, Err(err)) => Err(err),
            }
        }
        _ => Err(self.peek_invalid_type(&visitor)),
    };

    match value {
        Ok(value) => Ok(value),
        Err(err) => Err(self.fix_position(err)),
    }
}
```

## Test with serde_test

Serde는 custom serializers, deserializers에 대한 unit test를 지원하는 `serde_test` 모듈도 지원한다.

위에서 만든 예시를 테스트할 수 있는 코드는 다음과 같이 쓸 수 있다.

```rust
#[test]
fn test_ser_de() {
    use serde_test::{Token, assert_de_tokens};
    let c = KubeConfig {
        port: 10,
        healthz_port: 11,
        max_pods: 12,
    };

    assert_de_tokens(
        &c,
        &[
            Token::Struct {
                name: "KubeConfig",
                len: 3,
            },
            Token::Str("port"),
            Token::U8(10),
            Token::Str("healthz_port"),
            Token::U8(11),
            Token::Str("max_pods"),
            Token::U8(12),
            Token::StructEnd,
        ],
    );
}
```
`assert_de_tokens()` 함수는 deserializer를 테스트할 때 쓴다. enum `Token`을 정의해놓고 JSON처럼 구성한 후 이를 비교한다. `KubeConfig` struct를 만들 때 derive(PatialEq)을 집어넣은 이유가 여기서 사용하기 위해서다.
