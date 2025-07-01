---
layout: post
title:  "Optimized VM translator"
date:   2025-06-28 00:57:27 -0400
categories: nand2tetris
---

## A Hello World in < 16K instructions

TLDR: When compiling VM code to assembly, it is most efficient to wrap the code into groups that are either stack neutral or stack + 1, then optimize for every possible scenario in each type of group.


If you've ever tried your hand at writing a Jack to Hack compiler for nand2tetris, you'd face an immediate problem. The ROM only supports at most 32k instructions. Writing complex software with an accompanying operating system library with this limitation would be difficult even in a modern assembly language, but HACK assembly has its own unqiue challenges. There is only one general purpose register - the **data** (D) register, while the **address** (A) register is used to reference either the ROM (when jumping, we jump to the instruction at ROM[A]) or RAM (the value at RAM[A] is also known as the **memory** (M) register).

There is no bitwise NOR, NAND, XOR, XNOR, no bit shifting of any kind, and no multiply or divide. The only constants we have direct access to is -1, 1 and 0. The only constant math we can do is an increment or decrement by one. But worst of all, there is no stack implemented in hardware. Instead, it needs to be emulated in software, with RAM[0] representing the top of the stack - the actual data at the top of the stack is the data right before it. This means that pushing and popping from and to the stack and the D register are non trivial operations.


{% highlight mkd %}
Push: 4 instructions
@SP
AM=M+1
A=A-1
M=D

Pop: 3 instructions
@SP
AM=M-1
D=M
{% endhighlight %}

There is no push or pop assembly instruction, which means that anyone who wishes to squeeze as much space as possible needs to avoid the stack like the plague. Take the following example

{% highlight mkd %}
push local 0
push constant 1
add
pop local 0
{% endhighlight %}


If we were the tranlsate this VM code line by line, it would take 11 instructions

{% highlight mkd %}
@LCL
A=M
D=M
@SP
AM=M+1
A=A-1
M=D //push local 0
@SP
AM=M+1
A=A-1
M=1 //push constant 1
@SP
AM=M-1
D=M
A=A-1
M=D+M //add
@SP
AM=M-1
D=M
@LCL
A=M
M=D //pop local 0
{% endhighlight %}



Yet, this can be achieved in just 3 instructions

{% highlight mkd %}
@LCL
A=M
M=M+1
{% endhighlight %}


While this is a somewhat contrived example, it speak the core idea about writing optimal code. Our assembly is meant to be a reflection of our high level code. When writing high level code, we don't think in terms of pushes and pops; we think about expressions and assignments. So when parsing our VM code, we should identify groups that are either expressions, where the net stack count is +1, or assignments, where the net stack count is neutral. While I certainly have my complaints with HACK, there is one benefit makes this goal possible. In HACK, there is no "load" or "store" instruction really. A single instruction can read, do a calculation and write to memory all by itself. Take the instruction "M=D+M" as an example. This code read from RAM, adds the value to D register, and writes to RAM, all without even acknowledging the stack's existance! Furthermore, RAM addresses 13, 14 and 15 are free to be used as general pupose registers as well. 


Since I used Java for my implemention, I thought in terms of interfaces and abstract classes. There exists an interface called "VMInstruction" which has just one method, the decode() method, whose job is the generate the proper assembly code for that instruction. There is also a "PushGroup" abstract class, which not only implements VMInstruction but also has another method, the setD() method, whose job is to set the D register to its value. A PushGroup represents code that is net stack +1, meaning it represents only a single value. There are 5 definintions of a Push Group, one trivial and 4 recursive. In essense, we are building a tree where the Push and Pop instructions (as well as other cores types of VM instructions) are the leaves, and the PushGroups and VMInstructions are the branches.

Our VM code is made up of Push and Pop Instructions, each of which contains an Address object. An Address can be trivial or non trivial, and also has a method which sets the M register to the address value, allowing us to use the D register as an accumulator. 

A PuhsInstruction is obviously a PushGroup. It doesn't even to use the D register unless it's segment is either local, argument, this or that and its index is at least 4, since we can use repeated A=A+1 to arrive at that address. Setting the D register for local 3 vs local 4:

{% highlight mkd %}
@LCL
A=M+1 //A is now local 1
A=A+1 //A is now local 2
A=A+1 //A is now local 3
D=M
{% endhighlight %}

{% highlight mkd %}
@4
D=A
@LCL
A=D+M
D=M
{% endhighlight %}


A similar trick is used for PopInstuctions (where we pop the value from the stack into some address), if it's address is trivial we use the D register to store the top's value. Else, we store it's address in RAM[13], then pop from the stack and jump to the address in RAM[13]. 

A UnaryPushGroup is a PushGroup followed by a unary operation, either 'not' or 'neg'. Trivial, we can just call setD on the internal PushGroup, and then do D=-D or D=!D. However, if the internal PushGroup is also a PushInstruction, we can set the A register to the address' value, and then do either D=-M or D=!M, saving an instruction. There are also optimizations regarding double nots/negs as well and consecutive not/neg and neg/not, but those are better caught when parsing rather than evaluating.

A BinaryPushGroup is two PushGroups - a left and a right - with a binary op. While not that much more complicated than a UnaryPushGroup, we have to deal with the following situations:

1. Where the left and the right are the same

2. Where the left and the right are nearby PushInstructions (local 0 and local 1 are just one A=A+1 away)

3. Where either the left or the right are constants (especially if either 1, 0 or -1)

4. Either the left or the right are trivial PushInstructions, meaning that we can arrive at their addresses without needing the D reigster for assistance.

5. Comparision instructions (more on this later)

All of this while keeping in mind that D=M-D and D=D-M represent two different operations. Can you tell I've had to debug some bad code?

Regardless, a simple soluation for non comparision operations is to just place the left's value on the stack using its decode method, set use the setD method on the right, and finally do the operation with D and M.

Decode:

{% highlight mkd %}
@SP
A=M-1
M=D+M //if an addition
{% endhighlight %}

SetD:

{% highlight mkd %}
@SP
AM=M-1
D=D+M //if an addition
{% endhighlight %}


A CallGroup is n PushGroups followed by a call to a function that takes n arguments, where n is at least zero. There's not that much optmization, except in the way we optimize a list of PushGroups (more on this later).


A Dereference is used to model a[b] in Jack - equivlant the *(a+b). Formerly, its defined as:

{% highlight mkd %}
PushGroup (also known as the base)
pop pointer 1
push that 0
{% endhighlight %}

The setD is not dificult to implement. Our goal is to set the A register to the value of the base, then do D=M. If the base is not a simple push constant x, then we use PushGroup's setD method, but replace the first character of the last line with an A instead of a D. If it is a simple constant, then we just do @ (the constant value).


# How I deal with comparision 

The VM code supports eq, gt and lt, which compares the top values on the stack. In each case, we are taking the difference between two values, and writting either false (0) or true (-1, all one's in two's complement). My solution is to have a global subroutine for doing this, where the return address is on the stack, and the D register is the difference. From then on, we simply use the base comparisions allowable in HACK

{% highlight mkd %}
(DO_GT)
@RETURN_TRUE
D;JGT 
@RETURN_FALSE 
0;JMP 
// want  (left == right)  ⇔ (D == 0)
(DO_EQ)
@RETURN_TRUE 
D;JEQ 
@RETURN_FALSE 
0;JMP 
// want  (left  < right)  ⇔ (D < 0)
(DO_LT)
@RETURN_TRUE 
D;JLT 
@RETURN_FALSE 
0;JMP 
// ---- set boolean in D --------------------------------------
(RETURN_TRUE)
D=-1 
@WRITE_BACK 
0;JMP 
(RETURN_FALSE)
D=0 
@WRITE_BACK 
0;JMP 
// ---- collapse stack and return -----------------------------
(WRITE_BACK)
@SP 
AM=M-1
A=M 
0;JMP 
{% endhighlight %}


# How I handle function calls and return

The VM code "call foo n" takes n arguments which are on the stack and returns the value of the function. All functions return a value; in Jack void functions return 0 which is instantly dismissed. When we call a function, we push its frame on the stack:

{% highlight mkd %}
push return-address
push LCL
push ARG
push THIS
push THAT
ARG = SP - n - 5
LCL = SP
goto f //Where f is the address of the function we are jumping to
(return-address)
{% endhighlight %}

Technically, we only need three pieces of information: the return address, the amount of arguments plus 5, and the address of the funciton we are jumping to. Luckily for us, we have access to RAM[13, 14 and 15] for our VM translator. We can think of these areas as the poor man's CPU registers. Therefore, we can set another label, where the amount of arguments plus 5 in R14, the calling function address in R13, and the return address in the D register. From then we, push the stack frame

{% highlight mkd %}
(CALL)
@SP 
AM=M+1 
A=A-1 
M=D 

@LCL 
D=M 
@SP 
AM=M+1 
A=A-1 
M=D 

@ARG 
D=M 
@SP 
AM=M+1 
A=A-1 
M=D 

@THIS 
D=M 
@SP 
AM=M+1 
A=A-1 
M=D 

@THAT 
D=M 
@SP 
AM=M+1 
A=A-1 
M=D 

@14 
D=M 
@SP 
D=M-D 
@ARG 
M=D 

@SP 
D=M 
@LCL 
M=D 

@13 
A=M 
0;JMP
{% endhighlight %}


The return code is even simpler

{% highlight mkd %}
(RETURN)

 -- FRAME = LCL // FRAME is a temporary variable

@LCL 
D=M 
@14
M=D 

-- RET = *(FRAME-5) // Put the return-address in a temp. var.
@5 
A=D-A 
D=M 
@15 
M=D 

-- *ARG = pop() // Reposition the return value for the caller

@SP 
AM=M-1 
D=M 
@ARG 
A=M 
M=D 

--SP = ARG+1 // Restore SP of the caller
@ARG 
D=M 
@SP 
M=D+1 

-- THAT = *(FRAME-1) // Restore THAT of the caller

@14 
A=M-1 
D=M 
@THAT 
M=D 

-- THIS = *(FRAME-2) // Restore THIS of the caller
@14 
A=M-1 
A=A-1 
D=M 
@THIS 
M=D 

-- ARG = *(FRAME-3) // Restore ARG of the caller
@14 
A=M-1 
A=A-1 
A=A-1 
D=M 
@ARG 
M=D 

-- LCL = *(FRAME-4) // Restore LCL of the caller
@14 
A=M-1 
A=A-1 
A=A-1 
A=A-1 
D=M 
@LCL 
M=D 

-- goto RET // Goto return-address (in the caller’s code)

@15 
A=M 
0;JMP 
{% endhighlight %}

# Consecutive Pushes

When declaring a function, we push the constant 0 __k__ times for __k__ local variables, which not only initializes our local variables but also allocates room for the local variables on the stack. Since cosntant 0 is extremely trivial, instead of running the decode method on each push instruction, we could just increment the stack pointer by k, then jump to the top and repeat "A=A-1, M=0" k times. In fact, there is whole rabbit of optimizations we can do with these pushes

For example, take this code in Ouptut.initMap:

{% highlight mkd %}
do Output.create(0,63,63,63,63,63,63,63,63,63,0,0);
{% endhighlight %}

Since we only have on non trivial push and a bunch of zeros, we can save a lot of instructions just by repeating our strategy, but with setting the D register to 63, and doing either M=0 or M=D , although we should be clear that we are doing top down, not down top. 


## Push Pop Pairs
A Push Group followed by a Pop Instruction is stack neutral, and a source of a lot of optimizations. 

1. A call group followed by pop temp 0. This is our VM trick to disregard the value from void functions. We could also just do @SP, M=M-1.

2. If the Push Group is really just a constant: either 0, 1 or -1. 

3. We are decrementing some value by one or by itself. This is our trick from before. For incrementing a value by itself:

{% highlight mkd %}
push argument 0
push argument 0
add
pop argument 0
{% endhighlight %}

becomes:

{% highlight mkd %}
@ARG
A=M
D=M
M=D+M
{% endhighlight %}

Our last optimization is if the pop has a trivial address - meaining we can arrive there without using the D register, we can save ourselves from using the stack. 

## Push Writers

The final piece of the puzzle is how we map writing to a derefenced address. In Jack, this would by something like : let a[b] = c; The pseudoVm code would be:


{% highlight mkd %}
push c      //First Push Group, the source
push a
push b
add         //Second PushGroup, the dest
pop pointer 1
pop that 0
{% endhighlight %}

This is stack netural, and it's decode it failry simple:

{% highlight mkd %}
    @Override
    public List<String> decode() throws Exception {
        List<String> asm = new ArrayList<>(dest.decode());
        asm.addAll(source.setD());
        asm.addAll(List.of("@SP", "AM=M-1", "A=M", "M=D"));
        return asm;
    }
{% endhighlight %}


# End Result

In the end, with my OS plus my Main.jack:

{% highlight mkd %}
/** Test program for the OS Screen class. */
class Main {
    function void main() {
        var String s;
        let s = "Hello, world!";
        do Output.printString(s);
        do Output.println();
        do s.dispose();
        return;
    }
}
{% endhighlight %}


The total assembly count is exactly 15999 (23177 if I didn't use any of the grouping optimizations and just compiled the VM code line by line)!

To be clear, there is sitll room for more. Not only did I not implement dead code elimination, I am certain there are implementation details in the respective PushGroups that I missed, not to mention changing the source code to be more space efficient. For example, in the course provided Output.jack, Output.init is the only function that calls Output.initMap. Wouldn't it be better to inline initMap into init, avoiding generating the code for the function call? I also didn't write any tests, so this is very much just me shooting from the hip here.


The full code is avaiable on my GitHub. It includes both a JACK compiler with additional features (multi dereferecing like a[b][c] and support for <= >= and ~=), the VM translator and my OS.

