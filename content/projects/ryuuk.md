+++
title="Ryuuk"
date="2017-05-11"
description="Ryuuk is a simple concurrent webserver written in C++17 and bsd sockects."
tags=["sfml", "c++", "networking"]
+++

Ryuuk is a concurrent web-server written in C++.

Currently Ryuuk runs only in POSIX compliant environments only.

## Building

**nix**
```
$ git clone https://github.com/amhndu/ryuuk
$ cd ryuuk
$ mkdir build && cd build
$ cmake ..
$ make -ji # where i = no. of cores you can spare
```

**Windows**   
~~Who uses windows anyway ?~~   
Currently not portable with windows. Contributions welcome.

## Why

* Becuase why not ?
* We wanted to learn a thing or two about sockets & HTTP

## Keikaku


#### Short term

~~* Basic HTTP 1.1 GET requests~~   
~~* A Functional web server~~   


#### Long term

* Most of HTTP/1.1
* World domination

For a more detailed keikaku, please see the [TODO](https://github.com/amhndu/ryuuk/blob/master/TODO.md)

## References

* https://tools.ietf.org/html/rfc723<0-7>  
* https://tools.ietf.org/html/rfc2616  
* http://www2005.org/cdrom/docs/p730.pdf  
