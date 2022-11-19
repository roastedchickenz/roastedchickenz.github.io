---
title: Microcorruption - Cusco
date: 2022-11-19 08:00:00 +7
categories: [computer]
tags: [ctf,microcorruption,msp430, assembly]
---

This is my writeup for microcorruption level [Cusco](https://microcorruption.com/debugger/Cusco)  
Let's just jump to it. As always, start with “b main”. 

![Login function](/public/microcorruption - cusco/1.png)  
The beginning is the same as the previous level, main function is replaced with the login function.

Looking at the login function, we see the alert “only 8-16 characters" again. This might suggest that we have to do another overflow stuff. We also should check the function “test\_password\_valid”.

![Inputting "password" to the input prompt](/public/microcorruption - cusco/2.png)  
As per usual, I input “password” for the prompt just to see what it does.

It's always useful to know where the input goes!  
![The input goes to the memory address 43e0](/public/microcorruption - cusco/3.png)  
Break to the function "test\_password\_valid" and scan through the Live Memory Dump to see where our input go. In this case, it went to the memory address 43e0.

## checking test\_password\_valid function

![Inside test_password_valid function](/public/microcorruption - cusco/4.png)  
Scanning through the function, there’s really nothing very interesting. It just move a bunch of stuffs around and do some operation.    
And again we encounter the same problem as last challenges, at instruction 4468 and 446c, the program calls the 0x7d interrupt function. From the lock [manual](https://microcorruption.com/public/manual.pdf):  
>0x7d takes two arguments. The first argument is the password that’s going to be checked and the second argument is the location of a flag to overwrite if the password is correct. 

Well, the problem is we don't know the password so we can't pass the password to the function's argument. 

## going back to login
I decided to go back to the login function and see if there’s something interesting.  
Well, the program checks on r15 and jumps to “password incorrect” if Zero and doesn’t jump if r15 isn’t zero and grants us access. 

So we need to make r15 not Zero or so I thought. After being stuck for hours debugging this, I realized that the sp (stack pointer) gets added by 10 at the end of the program (instruction 453a).  
Pay close attention to the stack pointer of 2 below images!  
![Before sp added by 10](/public/microcorruption - cusco/5.png)  
Above picture is before the stack pointer is added.

![After sp added by 10](/public/microcorruption - cusco/6.png)  
Above picture is after the stack pointer is added.

Seeing the location of the sp (stack pointer) after it’s being added, it all suddenly clicked. Also, for some reason I forget that we have to do some overflow, so that's not good.

After being added, Sp (stack pointer) ends on the 17th character of our input and there’s a return coming right up, meaning the sp (stack pointer) will be called to tell where the next instruction will be executed after this. I wish i noticed this sooner ;_;  
Looking through the login function again, we see the function "unlock_door"!  
Lets try to redirect the program to that function. In this case we want the program to go to the memory address 4644.
 
## Final answer

The final payload should look something like (don’t forget little-endian and checks hex input option):  
**4141 4141 4141 4141 4141 4141 4141 4141 4644**

**Note: There's nothing special about hex "41", what's important is the amount of arbitrary characters must be 16 character and the 17th character must be the address of the "unlock_door function"**  

Below is what it looks like at Live Memory Dump after we overflow the memory.
![After sp added by 10](/public/microcorruption - cusco/7.png)  
When the return is called, it will look at the stack pointer and the stack pointer will tell the program what the next instruction should be. And since we set the 17th character to the memory address 4644, it will go that address and execute whatever in it. Which in this case is the unconditional "unlock_door" function.

Done 😉️  

![Done](/public/microcorruption - cusco/8.png)  
