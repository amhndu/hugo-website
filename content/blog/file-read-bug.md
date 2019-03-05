+++
title="The file you can't read on Windows"
date = "2017-02-04"
+++

I stumbled on this bug with my [NES emulator](https://github.com/amhndu/SimpleNES) where the ROMs won't open under Windows but had worked without a hitch on Linux and macOS.  
I had been developing on Linux and occasionally it tested on macOS. So it wasn't until quite late into the development that I found that the emulator can't open _any_ ROM files on Windows.  
This bug had me really perpexled, for why the emulator breaks only under Windows.  
My immediate reaction was of course to suspect Microsoft, as the old saying goes “Whenever anything goes wrong, always blame Microsoft.”  
Below is the relavant part of the code:
  
```cpp
bool Cartridge::loadFromFile(std::string path)
{
    std::ifstream romFile (path);
    ...
    std::vector header(0x10);
    if (!romFile.read(reinterpret_cast(&header[0]), 0x10))
    {
        LOG(Error) << "Reading iNES header failed." << std::endl;
        return false;
    }
    ...
} 
```
  
No matter what ROM you tried, the above function always failed and returned early as above.
However, if you try to open any other binary file (which is not an NES ROM) with the template I used above, it would at least go past this early check, thus I realized the key has to be the iNES file format itself.  
So I looked up at the hexdump
```
$ xxd nestest.nes | head
00000000: 4e45 531a 0101 0000 0000 0000 0000 0000  NES.............
00000010: 4cf5 c560 78d8 a2ff 9aad 0220 10fb ad02  L..`x...... ....
00000020: 2010 fba9 008d 0020 8d01 208d 0520 8d05   ...... .. .. ..
00000030: 20ad 0220 a220 8e06 20a2 008e 0620 a200   .. . .. .... ..
00000040: a00f a900 8d07 20ca d0fa 88d0 f7a9 3f8d  ...... .......?.
...
```
Do you spot something ?  
  
So after some futile attempts, I tried a hunch and much to the bewilderment of me and the friend who was helping me debug this on Windows, it worked!  
  
The patch:
```cpp
std::ifstream romFile (path, std::ios_base::binary | std::ios_base::in);
```
Yup, that's all it was, all I lacked was the binary flag.  
But, this didn't explain why this only happened with Windows nor why I had never encountered this issue while working on other binary files before.  
After some searching found something interesting on [cppreference](http://en.cppreference.com/w/cpp/io/c#Binary_and_text_modes), quoting the important parts:  
     “..on Windows OS, the character `\0x1A` terminates input.”  
     “POSIX implementations do not distinguish between text and binary streams (there is no special mapping for \\n or any other characters)”  
  
Look back at the first line.
```
00000000: 4e45 531a 0101 0000 0000 0000 0000 0000  NES.............
```
