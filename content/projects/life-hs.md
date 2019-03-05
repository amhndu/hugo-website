+++
title="life.hs"
date="2017-01-12"
description="A dead-simple Conway's Game of Life simulation written in Haskell."
+++

A dead-simple simulator for Conway's Game of Life in Haskell.

<script id="asciicast-JhfMNFYTlr4SdtDFvOO2yFecv" src="https://asciinema.org/a/JhfMNFYTlr4SdtDFvOO2yFecv.js" async></script>

Compiling:
```
$ ghc --make life.hs
```

Usage:
```
./life < seeds/PulsarSeed.txt
```
Or
```
$ runhaskell life.hs < seeds/GliderSeed.txt
```

Seeds are space separeted matrices of 0 and 1 representing dead/alive cell.
Note that it uses ANSI escape sequences so this may not work on Windows.

