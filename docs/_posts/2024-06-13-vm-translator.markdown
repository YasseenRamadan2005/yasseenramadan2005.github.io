---
layout: post
title:  "Documentation for my VM Translator (Part One) - nand2tetris Project 7"
date:   2024-06-13 00:57:27 -0400
categories: nand2tetris
---



# Intro


I decided to abandon Lua since it doesn't have switch statements or continue, in favor of Python, which I'm more familiar with. The core idea of the code is that VM code is more efficient ***grouping*** than to convert each line individually into assembly.


# Reachable vs Unreachable addresses

Before we begin, we have to understand the difference between a ***reachable*** and ***unreachable*** address. For an address in the form "segment index", we can set the A register to that addresses like:

{% highlight mkd %}
@index
D=A
@segment //Segments are addreses that contain pointers
A=D+M    //So they contain a pointer to another piece of memory
{% endhighlight %}



As you can see, I have to override the D register just to get to the address. For the command "push segment index", I need to arrive to the top of the stack with the value at "segment index". 



{% highlight mkd %}
@index
D=A
@segment //Segments are addreses that contain pointers
A=D+M    //So they contain a pointer to another piece of memory
D=M
@SP
AM=M+1
A=M-1
M=D
{% endhighlight %}


The assembly for a pop instruction is simmilar. However, since I am transferring the value at the top of the stack to the address, I need to arrive at the address with the D register containing that top value. Luckily, the VM can use registers 13, 14, and 15 as temporary registers. We can think of them as the poor man's CPU Registers.


{% highlight mkd %}
@index
D=A
@segment
D=D+M
@13
M=D
@SP
AM=M-1
D=M
@13
A=M
M=D
{% endhighlight %}


As you can see, these are expensive instructions: nine instructions for a push instruction and twelve instructions for a pop instruction. The simplest optimzation is the most basic: not all segments are pointers. 

| Segment      | How to go to address |
| ----------- | ----------- |
| Constant      | Simply do @constant and then D=A (you can't pop to a constant). Even better if the constant is -1,0 or 1, since you can set the D register to that value regardless.       |
| Static   | Simply do @'NameOfTheFile'.index. The way the assembly store variables is how static variables are implemented.        |
| Pointer 0/1 | Pointer 0 is just @THIS and pointer 1 is @THAT. Then do M=D or D=M, since we are talking about the address's value, not the value the address points to. | 
| Temp | Simply do @(5+index) |


Even if the segment is a pointer (argument, local, this, or that), optimizations are still possible. After all, if the index is 0 or 1, simply @segment A=M(+1) is enough. For a push instruction, indexes larger than 3 would be more efficient to use the classic case, whilst smaller indexes save some instructions.

Classic push:


{% highlight mkd %}
@index
D=A
@segment //Segments are addreses that contain pointers
A=D+M    //So they contain a pointer to another piece of memory
D=M
@SP
AM=M+1
A=M-1
M=D
{% endhighlight %}


Push segment 3:

{% highlight mkd %}
@segment
A=M+1
A=A+1
A=A+1
D=M
@SP
AM=M+1
A=M-1
M=D
{% endhighlight %}

However, for a pop instruction, since I am using @13, I can use larger values of index and still save instructions, as long as the index is smaller than 8.

Classic pop:

{% highlight mkd %}
@index
D=A
@segment
D=D+M
@13
M=D
@SP
AM=M-1
D=M
@13
A=M
M=D
{% endhighlight %}

Pop segment :

{% highlight mkd %}
@SP
AM=M-1
D=M
@segment
A=M+1
A=A+1
A=A+1
A=A+1
A=A+1
A=A+1
A=A+1
M=D

{% endhighlight %}

In short, a reachable address is one that can be reached without even needing the D register at all, while an unreachable requires some trickery with my only data CPU register. In general, I'm going to define a reachable address as one where the index is less than 4 (though I have more leeway for pop instructions)

## Push-Pop pairs


A push instruction followed by a pop instruction simply copys the value at the push address to the pop address. Since each push and pop instruction increments and decrements the stack, a push instruction followed by a pop instruction is stack neutral, so I save instructions not even needing to reference the stack at all. To generalize, ***k*** pushes followed by ***k*** pops is stack neutral, where the nth term is grouped with the k-n+1th term. For example:

{% highlight mkd %}
push 1
push 2
pop 3
pop 4
{% endhighlight %}

translates to 

{% highlight mkd %}
push 1
pop 4

push 2
pop 3
{% endhighlight %}


If there are more pushes than pops, or vica-versa, then those extra are grouped individually. This is the core idea of my code, VM code that's grouped can be translated more efficiently than code that's grouped individually.

For a push/pop pair, I simply have to deposit the value at the push instruction into the pop's address. If I'm depositing a constant, great! (Even better if pushing -1,0 or 1, since we have access to these constants in the assembly already). If I'm popping to a reachable address, great!, since I don't need to overwrite the D register to get there. If the pop address is unreachable, I can first store the address @13, then collect the push's value into the D register, and deposit wherever the pointer @13 takes me. There is also a concept I came up of mutally reachably addresses, whereby if you reach one it's trivial to reach the other (such as push local 500 followed by pop local 501, you simply need to increment the A register to get to 501 if at 500), but I decided not to implement that for now.


# Math instructions

The VM contains the following math instructions:
- Basic 2 input: add, sub, bitwise and & or
- Basic 1 input: bitwise neg, not
- Boolean compare: eq, lt, gt

The boo0lean compare instructions are complicated to implement, since its outside the provided assembly commands, so I'll discuss them later. I'm going to call a basic math instruction a 'Math instruction', a 2 input instruction a 'math2' and a 1 input instruction a 'math1'.


The obvious optimzation is to group code in the form:

{% highlight mkd %}
push 1
push 2
math2
{% endhighlight %}


or 

{% highlight mkd %}
push
Math
{% endhighlight %}

However, we can go further. For example, take the high-level code x=x+1, where x is the first local variable. The vm code would be

{% highlight mkd %}
push local 0
push constant 1
pop local 0
{% endhighlight %}

while the most optmized assembly code would be

{% highlight mkd %}
@LCL
A=M
M=M+1
{% endhighlight %}


Clearly, this is an extreme example, but it leads to a major discovery. If we take a basic math group (with either 2 pushes and a math2 or one push and a Math), then we can write more optimized code if the next instruction is either a pop or Math instruction. math1 instructions are stack neutral, since they only modify the top value on the stack, while math2 instruction decrement the stack.

## Set the D register to the basis

Suppose we have some VM code either 

{% highlight mkd %}
push 1
push 2
math2
{% endhighlight %}


or 

{% highlight mkd %}
push
Math
{% endhighlight %}

It would be very helpful if we can set the D register to the correct value of that code. To start with, any constants can be dealt with manually, so "push x, push y, add" is the same as @(x+y), D=A. Adding by 1 and subtracting by 1 can also be simply dealt with - just go to the other address and decrement/increment. 


If I'm then popping to the same address I'm working with, I can simply stay on that address. For example:

{% highlight mkd %}
push s1
push s2
sub
pop s2
{% endhighlight %}

would be 

{% highlight mkd %}
(Store address of s2 @13 if needed)
(Withdraw the value from s1 into the D register)
(Go to s2)
M=D-M
{% endhighlight %}


You have to be careful with subtraction. As the only non communative operater, the code for 

{% highlight mkd %}
push s1
push s2
sub
pop s1
{% endhighlight %}

would instead be

{% highlight mkd %}
(Store address of s1 @13 if needed)
(Withdraw the value from s2 into the D register)
(Go to s1)
M=M-D
{% endhighlight %}

Another optimization is when adding a number to itself (very helpful for multiplication)

{% highlight mkd %}
push s1
push s1
add
pop s1
{% endhighlight %}


would be 

{% highlight mkd %}
(Go to s1)
D=M
M=D+M
{% endhighlight %}


I'm oversimplifying a little. For example, the code 

{% highlight mkd %}
push s1
add
add
{% endhighlight %}

is decrementing the stack pointer by 1. while

{% highlight mkd %}
push s1
push s2
add
add
{% endhighlight %}

is stack neutral. For the basis where I'm working with one push and a math2 instruction, I need to go the stack to grab the next element. If there's no more instructions, then I simply dump the value there. So 


{% highlight mkd %}
push s1
add
neg
{% endhighlight %}

is 


{% highlight mkd %}
(Get value from s1)
(Go to top of stack)
M=D+M
M=!M
{% endhighlight %}

while 

{% highlight mkd %}
push s1
add
pop s2
{% endhighlight %}

would be

{% highlight mkd %}
(Get value from s1)
(Go to top of stack)
**D=D+M**
(Then deposit value to s2)
{% endhighlight %}


{% highlight mkd %}
push s1
add
add
{% endhighlight %}

is

{% highlight mkd %}
(Get value from s1)
@SP
AM=M-1
D=D+M
A=A-1
M=D+M
{% endhighlight %}


I'm not going to sugercoat this, it's ALOT of spaghetti code to deal with each case. But it saves instructions, and that's all I care about!


# Boolean Instructions

The boolean instructions eq, lt and gt, with true being defined as -1 (all ones) and false as 0 (all zeros). For each case, I take the difference of the top two stack values, and return either -1 or 0 based off the logic. Here's the starting subroutine for my code, assuming that @14 contains either -1, 0 or 1 if doing lt, eq, or gt.

{% highlight mkd %}
@SKIP
0;JMP //Start off skipping

(COMP_BEGIN)
@13
M=D //Store the return address @13
@SP
A=M-1
D=M
A=A-1
D=M-D //Calculate the difference

@15
M=D //Store the difference at R15
@14 //Check the flag @14
D=M
@EQ_BEGIN
D;JEQ   // Jump to the equal check if 0
@LT_BEGIN
D;JLT // Jump to the lt check if -1

//By this case, this is the greater than subroutine
@15
D=M
@RETURN_TRUE
D;JGT
@RETURN_FALSE
0;JMP

(EQ_BEGIN)
@15
D=M
@RETURN_TRUE
D;JEQ
@RETURN_FALSE
0;JMP

(LT_BEGIN)
@15
D=M
@RETURN_TRUE
D;JLT
@RETURN_FALSE
0;JMP

(RETURN_TRUE)
D=-1;
@COMPLETE
0;JMP

(RETURN_FALSE)
D=0;
@COMPLETE
0;JMP

(COMPLETE)
@SP
AM=M-1
A=A-1
M=D
@13
A=M
0;JMP
(SKIP)
{% endhighlight %}

Here's how I would call it for equality:

{% highlight mkd %}
@RETURN.FROM.COMP.OPS.0
D=A
@14
M=0
@COMP_BEGIN
0;JMP
(RETURN.FROM.COMP.OPS.0)
{% endhighlight %}


Since there's a custom return address, I create a global variable to track where I'm returning to. There are some optimzations I can do with constants but this is enough for now.



# Summary:

BasicTest - 76 instructions
PointerTest - 46 instructions
StaticTest - 24 instructions
SimpleAdd - 5 instructions
StackTest - 238 instructions (yeah yeah I know optimizations with constants would make it run better).
