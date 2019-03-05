+++
title = "Let's Make an Extremely Readable Birthday Melody, the IOCCC Way"
date="2017-04-30"
+++

**Obfuscate:** tr.v. -cated, -cating, -cates.  
1.  i.  To render obscure.  
    ii.  To darken.
2.  To confuse: his emotions obfuscated his judgment.
    \[Lat. obfuscare, to darken : ob(intensive) + Lat. fuscare,
    to darken < fuscus, dark.\] -obfuscation n. obfuscatory adj

(taken from [IOCCC](http://www.ioccc.org/))

### The Why

1.  You learn lesser known aspects of the language
2.  Job security. Write obfuscated code and make sure no one else can maintain it but you!
3.  It's fun. 'nuff said.

### The Plan

We'll be making a small C program which generates [samples](https://en.wikipedia.org/wiki/Sampling_(signal_processing)) of a song (we'll stick with the 'Happy Birthday' song), whose output will then be piped to aplay. Most standard Linux distributions come with aplay, bundled along with ALSA (default on most Linux distributions). To follow along on macOS or Windows, you might try using [sox](http://sox.sourceforge.net/).

We'll start with a non-obfuscated version, and obfuscate it, step-by-step.

### The Code

Here's the base we'll be obfuscating. Although it's commented, understanding the sample generation isn't very important as long as you see the big picture.

```cpp
#include <math.h>
#include <stdio.h>

int main()
{
    int frequencies[]={0, 392, 440, 493, 523, 587, 659, 698, 783}, i, j;
    /* The Song.
    * @ maps to zero frequency, it comes right before A in ASCII, this makes mapping these notes to the frequency index trivial.
    */
    char song[] = "AABADC@AABAED@AAHFDCB@HHGDED";
    double sample_rate = 8000,
        amplitude = 128,
        duration  = 400;   // in ms

    for (j = 0; j < sizeof(song)/sizeof(song[0]); ++j)      // For each note in the song
    {
        for (i = 0; i < sample_rate * duration / 1000; ++i) // Print a sample
        {
            // Fade in/out sinusoidally
            double amp = amplitude * sin(M_PI * (i / (sample_rate * duration / 1000.0)));
            /*
            * i samples   =>  (i / sample_rate) seconds
            * 1 second    =>  note_frequency wavelengths
            * Thus, (i / sample_rate) seconds =>
            *       wavelengths = ((note_frequency * i) / sample_rate)
            * And, sample = sin ( 2 * PI * wavelengths)
            */
            int frequency = frequencies[song[j] - '@'];
            char sample =  127 + amp * sin((2.0 * M_PI * i * frequency) / sample_rate);
            printf("%c", sample);
        }
    }
    return 0;
}
```
Run this (on Linux):

    gcc unobfuscated.c -w -lm && ./a.out | aplay -f U8 -r 8000

If you're using sox:

    gcc unobfuscated.c -w -lm && ./a.out | play -c 1 -b 8 -e unsigned -traw -r 8k -

You should be able to hear a nice little "Happy Birthday To You" melody.

### The Obfuscation

First, let's substitute those space-wasting variables with [magic numbers](https://en.wikipedia.org/wiki/Magic_number_(programming)). Then, we can just remove the include for stdio, it's still valid code. The compiler will assume the implicit definition for printf with return type int. And during linking, the actual printf can just fit in here! However, we can't do this with math.h, as the sin() function returns a double, which will cause a linker error.
In C, all global variables or functions without a given type are implicitly assumed to be int (or in case of functions, return type is int as remarked above).

This let's us do
```cpp
    _[]={0,392,440,493,523,587,659,698,783},i,j;
    main()
    ...
```

Another added benifit of glabal variables is that they are automatically initialized to 0. I also renamed the frequencies array to \_. This greatly _improves_ readability.

Here's our inner loop:
```cpp
for (i = 0; i < 3200; ++i)
{
    char sample =  127 + amplitude * sin(9.81e-4 * i) * sin((7.85e-4 * i * _[song[j] - '@']);
    printf("%c", sample);
}
```
We'll use another small C quirk.

`A[i]` is equivalent to `i[A]` where A can be any array-type or a pointer.
The array subscript is just syntactic sugar for pointer arithmatic. `A[i]` is essentially converted to `*(A + i)`.
Note that if you swap A and i in this form, it doesn't really make a difference. Instead of A here, we can put our song, which is a string.
The string will decay to a pointer to the first element and we'll subsequently get the corresponding note. Also, we see that @ has an ASCII value of 64, a power of 2.
Observe that the minuend is always between 64 and 64 + 8 (eight is the total number of frequencies we can play). Instead of subtracting by 64, we can simply mask off the four lower bits to get the index. So,
```cpp
int frequency = frequencies[song[j] - '@'];
```
becomes
```cpp
_[j["AABADC@AABAED@AAHFDCB@HHGDED"]&15
```
For the next obfuscation step, let's look at the program flow
```cpp
main()
{
    for (j = 0; j < 28; ++j)
    {
        for (i = 0; i < 3200; ++i)
        {
            ...
        }
    }
    return 0;
}
```
We can instead just use one loop and whenever we hit the end condition for i, we reset i and increment j. Something like:
```cpp
main()
{
    for (; j < 28; ++i)
    {
        ...
        if (i >= 3200)
        {
            i = 0;
            ++j;
        }
    }
}
```
To make our code more _readable_, let's just use recursion. On main.
```cpp
#include <math.h>
_[]={0,392,440,493,523,587,659,698,783},i,j;
main()
{
    char __ =  127 + 128 * sin(9.81e-4 * i) * sin((7.85e-4 * i * _[j["AABADC@AABAED@AAHFDCB@HHGDED"]&15]));
    printf("%c", __);
    if (i >= 3200)
    {
        i = 0;
        ++j;
    }
    if (j < 28)
    {
        ++i;
        return main();
    }
    else
        return 0;
}

```
Those if blocks are quite verbose. Let's replace them with something more fun.
Real programmers use [short circuit evaluation](https://softwareengineering.stackexchange.com/a/201899),
not the ternary conditional operators or measerly if blocks. Here's quick revision on short circuit evaluation:

1.  `a && b` --> b will only be evaluated if a is true. If a is false, we know the expression will be false anyway.
2.  `a || b` --> b will only be evaluated if a is false. If a is true, the expression will be true regardless of b.

Talking about fun things reminds of the less-used [comma operator](https://stackoverflow.com/a/52558).
It lets us do (expression1, expression2), which will evaluate to expression2.
How else can you code more _readable_ than doing many things in one line ?

Another useful operator is the xor operator. It has some useful properties that we'll be using. For any integers _a, b_, the following hold:

1.  `a ^ a` == 0
2.  Thus, `a != b` is equivalent to a ^ b (testing equality)
3.  `a ^= a` will zero a number. (In fact, this is a very common optimization, used by compilers and assembly programmers alike)


Putting it all together:
```cpp
#include <math.h>
_[]={0,392,440,493,523,587,659,698,783},i,j;
main()
{
    char __ =  127 + 128 * sin(9.81e-4 * i)
            * sin((7.85e-4 * i * _[j["AABADC@AABAED@AAHFDCB@HHGDED"]&15]));
    printf("%c", __);
    i++^3200 || (i^=i,++j);
    return (j^28) && main();
}
```
Now this looks like some obfuscation! But we're not done yet.

Instead of printf, we can use [write(2)](https://linux.die.net/man/2/write). If you were following on Windows, then I'm sorry you won't be able to follow this step.

The printf line becomes:

    write(STDIN, &__, 1);

Now we can strip all whitespace and make some minor tweaks to get this final version (run this the same way we ran the first unobfuscated version):
```cpp
#include <math.h>
_[]={0,392,440,493,523,587,659,698,783},i,j;
main(){char __=127+(1<<7)*sin(9.81e-4*j)*sin
((7.85e-4*j*_[i["AABADC@AABAED@AAHFDCB@HHGD\
ED"]&017]));write(__LINE__>>0x2,&__,~*_&1);(
j++^3200)||(j^=j,++i);return(i^28)&&main();}
```
The following are the tweaks (most are to justify the text, some increase the line-width, some decrease):


1.  Replaced 128 with `(1 << 7)`. See [Bit shifts](https://en.wikipedia.org/wiki/Bitwise_operation#Bit_shifts).
2.  Replaced 15 with 017. Literals starting with a zero (not followed by an x) are in octal.
3.  STDIN's fd is 1. The write call is in line 5, thus `__LINE__ >> 2` evaluates to 5. As evident from the name, the macro `__LINE__` expands to the line number.
4.  For write()'s third arguemnt, we use the incantation `~*_&1`
    First, `_` is the frequencies array, decayed to the base pointer. We dereference it, thus we get the first element, which is 0. Then we bitwise-negate it (~), we get a whole bunch of 1s which we finally bitwise-and with 1 which just yields 1. This all works out thanks to the [C precedence rules](http://www.difranco.net/compsci/C_Operator_Precedence_Table.htm).



And there you have it! Wish your loved one a very happy birthday, the IOCCC way ;)
