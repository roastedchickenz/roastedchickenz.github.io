---
title: Microcorruption - Hanoi
date: 2022-08-29 10:00:00 +7
categories: [computer]
tags: [ctf,microcorruption,msp430, assembly]
---

This is my writeup for microcorruption level [Hanoi](https://microcorruption.com/debugger/Hanoi)

Let's just jump to it. As always, type “b main”. 

![after "b main"](/public/microcorruption - hanoi/1.png)

It looks like main just calls a function called login which includes a bunch of interesting things, the login function now effectively becomes our new main function.  
I’m gonna “unbreak main” function and “break login”.

![inside login function](/public/microcorruption - hanoi/2.png)

Scanning through the login function, we see a few interesting things, first, an alert telling us to input a password between 8-16 characters (what would happen if we enter more than 16 characters 😉️ ) and a function called “test\_password\_valid”, interesting let's take a look.  
For the input prompt, I enter “password” just to see where it stores our input and maybe we can do something with it. 

![input "password" when interrupted](/public/microcorruption - hanoi/3.png)

## checking test\_password\_valid function
Type "b test\_password\_valid" and check what's inside the function.

![inside test_password_valid function](/public/microcorruption - hanoi/4.png)

While we're at it, take a second to notice that the string "password" we inputted earlier is placed at Live Memory Dump address 2400

![user input stored at memory address 2400](/public/microcorruption - hanoi/5.png)

Anyway, Looks like the "test\_password\_valid" function does some flag setting of some sort.Nothing really interesting.  
What caught my eye was the instruction 446a and 446e, it calls the 0x7d interrupt. From the lock [manual](https://microcorruption.com/public/manual.pdf):  
>0x7d takes two arguments. The first argument is the password to test, the second is the location of a flag to overwrite if the password is correct. 

Well, the problem is we don’t know the password so this is not what we're looking for. 

## actually solving it

![going back to login function after checking "test\_password\_valid" function](/public/microcorruption - hanoi/6.png)

We see after the test\_password\_valid, the program test for r15, and set a hex value 0x36 to address 2410 if r15 isn’t Zero. looking down a bit more we see instruction 455a which compares the hex 0x8e with the value at address 2410. Okay, i’m not sure why the program sets the value as 0x36 if at the end they check for hex value 0x8e, my first guess was 0x36 is the flag set if we enter the correct password but apparently it doesn’t matter since it checks for 0x8e.  

After the program sets 0x8e, it then checks if 0x8e exists in memory address 2410, if 0x8e exist then we get access granted otherwise no access. 

So, we need to somehow fill memory address 2410 with hex value 0x8e. How to do just that?  
Remember earlier we saw the password we enter appear at memory address 2400 and we should have a hex value at memory address 2410? that's awfully close!  
And remember earlier we were told to enter password between 8 to 16 characters?  
Now we know why they want us to input only 8 to 16 characters! Because the lock makes a check at address 2410 to determine if we inputted the correct password or not.

## Final answer

What we need to do is overflow the password input and the 17th character must be 0x8e. So our final input should be (don’t forget the little-endian and checks hex input option): **4141 4141 4141 4141 4141 4141 4141 4141 8e00**  

**Note: There's nothing special about hex "41", what's important is the amount of arbitrary characters must be 16 character and the 17th character must be 0x8e**  

Below is what it looks like at Live Memory Dump after we overflow the memory.

![overflowing the input and 0x8e exists at address 2410](/public/microcorruption - hanoi/7.png)

Done! 😉️

![door unlocked](/public/microcorruption - hanoi/8.png)





























