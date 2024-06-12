---
layout: post
title:  "Documentation for my HacK Assembler - nand2tetris Project 6"
date:   2024-06-12 00:57:27 -0400
categories: nand2tetris
---

### Intro
I decided to write [the assembler](https://github.com/YasseenRamadan2005/HACK-assembler-implentation-in-Lua/) in Lua after wathcing [a fireship video explaining the language](https://www.youtube.com/watch?v=jUuqBZwwkQw). There are two types of instruction the assembler deals with: A-Instructions and C-Instructions.


#### C-Instructions
C-Instructions work on the ALU to do basic math - divided into three parts: {% highlight mkd %}
dest=comp;jump
{% endhighlight %}


 With only 

- 3 registers as the destination (with any combination possible)
- Only addition, subtraction, bit-wise AND/or (must be the D register with either the A or M register)
- Only constants for the comp being 0, -1 and 1 (+1 and -1 are allowed, while bitwise operations on constants are not)
- And only eight possible jump instructions (including no jump)

it means there are only a finite (though numerous amount) of C instructions. Instead of writing a function that converts the C instruction into the 16-bit machine code like a "normal" programmer, I decided to whip up some python code to create a large look-up table for the C-Instructions, thus creating the only (as far as I know) complete list of possible C instructions for the HACK architecture

#### A-Instructions



### Contact

- School Email: ywr5999@rit.edu
- Personal Email: yasseenramadan6@gmail.com
- Linkedin: [linkedin.com/in/yasseen-ramadan-4590a1286](www.linkedin.com/in/yasseen-ramadan-4590a1286)
- Github: [github.com/YasseenRamadan2005](https://github.com/YasseenRamadan2005)
