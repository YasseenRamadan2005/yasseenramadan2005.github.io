---
layout: post
title:  "Documentation for my Hack Assembler - nand2tetris Project 6"
date:   2024-06-12 00:57:27 -0400
categories: nand2tetris
---

## Intro
I decided to write [the assembler](https://github.com/YasseenRamadan2005/HACK-assembler-implentation-in-Lua/) in Lua after wathcing [a fireship video explaining the language](https://www.youtube.com/watch?v=jUuqBZwwkQw). There are two types of instruction the assembler deals with: A-Instructions and C-Instructions.


## C-Instructions
C-Instructions work on the ALU to do basic math - divided into three parts: {% highlight mkd %}
dest=comp;jump
{% endhighlight %}


 With only 

- 3 registers as the destination (with any combination possible. So A=M+1 is valid. So is AM=M+1, AMD=M+1, MD=M+1 and AD=M+1)
- Only addition, subtraction, bit-wise AND/or (must be the D register with either the A or M register)
- Only constants for the comp being 0, -1 and 1 (+1 and -1 are allowed, while bitwise operations on constants are not)
- And only eight possible jump instructions (including no jump)

it means there are only a finite (though numerous amount) of C instructions. Instead of writing a function that converts the C instruction into the 16-bit machine code like a "normal" programmer, I decided to whip up some python code to create a large look-up table for the C-Instructions, thus creating the only (as far as I know) complete list of possible C instructions for the HACK architecture

## A-Instructions


Any instruction that begin with a '@' is an A-Instruction. The instruction select a specfic value of the RAM/ROM. There are two types of A-Instructions: Trivial and non-Trivial.

### Trivial

- Any instruction in the form @number is translated to "0(number in binary)". So @1 is "0000000000000001". This included instructions in the form @R'number' where 'number' is from 0 to 15.

- Defualt translations defined here:

{% highlight lua %}
A_Instructions = {
    SP = 0,
    LCL = 1,
    ARG = 2,
    THIS = 3,
    THAT = 4,
    SCREEN = 16384,
    KBD = 24576
}
{% endhighlight %}


### Non-Trivial

For non-trivial A instructions, there are labels (for branch programming) and variables. Labels are delcared in the form (LABEL_NAME), which is not translated as an instruction itself, but instead the next line following (LABEL_NAME) is the start of the label. So a key-value pair of the label's name with the line number is added to the A-Instructions table. (When jumping, the Program Counter is set to the the A register). Since label declarations do not have to be declared in advance of the label's use, a first passs of collecting the labels is needed before anything else.


For variables, they start being allocated at RAM[16] and do not have to be declared. The code for this is simple, for instructin @thing, if thing is not in the table already, add it to the table with the value 16 + the amount of variables and increment the amount of variables.


## Code

I start off requring LuaFileSystem for chaning directories and Bitlib for bit hacking. Use luarocks install to install the dependencies.


### To16bitbinary

```lua
local function To16bitbinary(number)
    local bits = ''
    for i = 15, 0, -1 do
        bits = bits .. ((bit.band(number, bit.lshift(1, i)) ~= 0) and '1' or '0')
    end
    return bits
end
```

1. **Initialize an Empty String**:
    ```lua
    local bits = ''
    ```
    A local variable `bits` is initialized as an empty string. This will hold the resulting binary string.

2. **Loop Over Each Bit Position**:
    ```lua
    for i = 15, 0, -1 do
    ```
    A for-loop is set up to iterate from 15 down to 0. These numbers represent the bit positions in a 16-bit binary number, where 15 is the most significant bit (leftmost) and 0 is the least significant bit (rightmost).

3. **Bitwise Operations**:
    ```lua
    bits = bits .. ((bit.band(number, bit.lshift(1, i)) ~= 0) and '1' or '0')
    ```
    - `bit.lshift(1, i)`: This shifts the number `1` left by `i` positions, creating a bitmask where only the `i`-th bit is set to 1.
    - `bit.band(number, bit.lshift(1, i))`: This performs a bitwise AND between the input `number` and the bitmask. If the `i`-th bit in `number` is set, the result will be non-zero.
    - `((bit.band(number, bit.lshift(1, i)) ~= 0) and '1' or '0')`: This checks if the result of the bitwise AND is non-zero. If it is, it means the `i`-th bit in `number` is `1`; otherwise, it is `0`.
    - `bits = bits .. ...`: The result (`'1'` or `'0'`) is concatenated to the `bits` string.

4. **Return the Result**:
    ```lua
    return bits
    ```
    After the loop completes, the function returns the `bits` string, which now contains the 16-bit binary representation of the input `number`.


### Convert_Ainstruction

The Convert_Ainstruction function takes in parameter 'a' (yeah I know, amazing variable name). I use string.sub to extract the segment from the string @segment and tonumber to convert to an integer (which returns nil if not actually a number)

If it's an actual number, I pass it to another function, To16bitbinary, to create the machine code.

Else, if I can match a string in the form R(number), then I call tonumber after a string.sub for pass to To16bitbinary.

Else, if it's not in the A_Instruction table alreadysince its a variable, I increment the global tracker of the amount variables and add it to the table.

In the end, I return To16bitbinary with the value from the table.

### process_file

I start off changing the directory to input path. I then 'clean' the file by getting rid of the \r\n windows nonsence, as well as any spaces, as well as comments (in form //'comment'\n) as well as extra line characters. 

In my first pass, I collect the labels with the line numbers.

For my second pass, I convert all the non label declaration and write to the file.



## Conclusion

To be clear, the code only converts a singular file, instead of an entire directory of files. 

My largest speedups come from using the built-in string functions (which are written in ***very*** fast C) instead of trying to do it myself.
Luckily the translation is moslty one-to-one, so I don't have to optimze the underlying code. That's it for project 6!


### Contact

- School Email: ywr5999@rit.edu
- Personal Email: yasseenramadan6@gmail.com
- Linkedin: [linkedin.com/in/yasseen-ramadan-4590a1286](www.linkedin.com/in/yasseen-ramadan-4590a1286)
- Github: [github.com/YasseenRamadan2005](https://github.com/YasseenRamadan2005)
