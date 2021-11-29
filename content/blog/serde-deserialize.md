+++
title = "Writing custom serde deserializers, the easy way"
date = "2021-01-30T19:34:17+05:30"
tags=["rust", "serde", "deserialization", "serialization"]
+++

[Serde](https://serde.rs/) is a great library for serializing/deserializing in [Rust](https://www.rust-lang.org/). It allows you to directly convert rust data structures using a few attributes.
Most of the time, It Just Works &trade; and when it doesn't, you can write your own [serializers](https://serde.rs/impl-serialize.html) or [deserializers](https://serde.rs/impl-deserialize.html)!

Problem at hand: There exists a type which can be parsed from a string but serde has no built in support for it.
For the purposes of an example, let's say the type is `std::net::IpAddr` and *forget* that serde has support for it already.
This avoids us the distraction of creating a new type and parser.

Before jumping to the "easy" way, let's first follow the official documentation and try writing a deserializer. for the following struct:
```rs
struct Config {
    host: IpAddr,
    port: u16,
}
```

To use the [built-in parser](https://doc.rust-lang.org/nightly/std/net/enum.IpAddr.html#impl-FromStr), we need to build a [visitor](https://en.wikipedia.org/wiki/Visitor_pattern) that convers a string to our type.

```rs
struct AddrVisitor;

impl<'de> de::Visitor<'de> for AddrVisitor {
    type Value = std::net::IpAddr;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        formatter.write_str("a dotted-decimal IPv4 or RFC5952 IPv6 address")
    }

    fn visit_str<E: de::Error>(self, s: &str) -> Result<Self::Value, E> {
        s.parse().map_err(de::Error::custom)
    }
}

fn parse_addr<'de, D>(deserializer: D) -> Result<IpAddr, D::Error>
where
    D: Deserializer<'de>
{
    deserializer.deserialize_str(AddrVisitor)
}
```

Try it on the [playpen](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=edd1204037816233ec7556d5d8137726)

It's... quite a handful. A visitor may make sense if you want to support different types, but a lot of the times you really just want to convert between one of the base types (string / integer) to our domain type, which makes the flexibility of a visitor quite an overkill.


We can easily get rid of all this boilerplate with the following pattern:

```rs
fn parse_addr<'de, D>(deserializer: D) -> Result<IpAddr, D::Error>
    where D: Deserializer<'de>
{
    let addr_str = String::deserialize(deserializer)?; // <-- this let's us skip the visitor!
    addr_str.parse().map_err(de::Error::custom)
}
```

Try it on the [playpen](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4c4c21fe24ea0381b2d4d118cc56bdd0)

We went from 20 lines of boilerplate to 5 lines!
We leverage serde's existing string deserializer to first get the string, which can be readily `parse`d!
You can analogously use any of the base types serde supports and convert to the relevant domain type.

Reading the serde docs didn't make it obvious to me, so I hope this will be useful to somebody and save some boilerplate.

