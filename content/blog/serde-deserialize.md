+++
title = "Learn this one weird trick to write custom serde deserializers!"
date = "2021-01-30T19:34:17+05:30"
tags=["rust", "serde", "deserialization", "serialization"]
+++

[Serde](https://serde.rs/) is a great library for serializing/deserializing in [Rust](https://www.rust-lang.org/). It allows you to directly convert rust data structures using a few attributes.
Most of the time, It Just Works TM and when it doesn't, you can write your own [serializers](https://serde.rs/impl-serialize.html) or [deserializers](https://serde.rs/impl-deserialize.html)!

Except.. if you follow the official documentation, writing a visitor to implement a deserializer is very handful.

We can do better! With this one weird trick. (I'm sorry about the title, I couldn't think of a normal title) 

A lot of times you can just compose serde's existing deserializers much the same way it allows you to serialize.
This is especially useful if you already have a conversion from another type.
For the purpose of an example, let's try writing a serializer for `std::net::IpAddr`, assume for a moment serde doesn't support it out of the box.

```rs
#[derive(Deserialize)]
struct Config {
    #[serde(deserialize_with = "parse_addr")]
    host: IpAddr,
    port: u16,
}

fn parse_addr<'de, D>(deserializer: D) -> Result<IpAddr, D::Error>
where
    D: Deserializer<'de>,
{
    let addr_str = String::deserialize(deserializer)?;
    addr_str.parse().map_err(de::Error::custom)
}
```
Try it on the [playpen](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4c4c21fe24ea0381b2d4d118cc56bdd0)

We can re-use the implementation for `String` here, you can analogously use any of the base types serde supports by default!
