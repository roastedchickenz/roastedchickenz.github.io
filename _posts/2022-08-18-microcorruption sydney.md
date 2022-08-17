---
title: Microcorruption - Sydney
date: 2022-08-18 10:00:00 +7
categories: [computer]
tags: [ctf,microcorruption,msp430, assembly]
---

This is my writeup for microcorruption [Sydney](https://microcorruption.com/debugger/Sydney)


First, let's check our main function, go ahead and type “b main”.

![after "b main"](/public/microcorruption - sydney/1.png)

Scanning through the main function, it looks like the determinant to trigger the unlock is at instruction 4450 which will test if r15 is Zero or not. Then, instruction 4452 checks if Zero then the lock won’t be opened, if not Zero then the lock will be opened.

So, at first glance we should somehow make r15 not Zero. Since function "get_password" only take input from user and put it in the Live Memory Dump, nothing interesting happened there. Lets check the function check\_password. Type "b check\_password" to create breakpoint.

For now, I enter “password” when the prompt asks for input.
![enter "password" when prompted](/public/microcorruption - sydney/2.png)

Before we continue, take a second to notice that the "password" we entered appear in Live Memory Dump at address 439c
![enter "password" when prompted](/public/microcorruption - sydney/3.png)

## check_password function

Moving on,  
![enter "password" when prompted](/public/microcorruption - sydney/4.png)

We see a bunch of cmp statements (which is good, it means we could do something to fulfill the conditional statement).  
Taking a closer look at the "check_password" instructions, we see the program compare a constant hex character with whatever values located in r15 (conviniently, r15 is pointing to 439c, the location where our input is located as seen from screenshot above), and check if Zero (in other word, check if the value being compared is the same), the program does this 4 times!

Seems like this is another hardcoded password lock 😅️.  We could just enter the hardcoded password and it won’t be wrong right? (foreshadowing).

Let's test our theory that this is another hardocded password lock, reset the lock (by typing "reset") and enter the hardcoded password **7a3e 7c62 577a 7725**. Don't forget to check the Hex input option since we’re entering hex. 
![enter hardcoded password](/public/microcorruption - sydney/5.png)

Running the rest of the code and we get...
![Invalid password](/public/microcorruption - sydney/6.png)

Invalid password 🤪️.

Not gonna lie, the first time I encountered this, I’m pulling my hair like a madman, I even checked my input a few times to make sure I enter the correct value. So, What did I do wrong? Well, the answer is because of a thing called “little-endian”. 
## Endian [Little-endian & Big-endian]
What is little-endian? Basically, some computers store information a bit differently than us humans. Notice the word “some”.  
Little-endian computers store the password in “reverse”. So if we take our example “7a3e 7c62 577a 7725” each compare character should be reversed. So our final answer should look something like “**3e7a 627c 7a57 2577**” notice that the order of the compares stays the same, only each compare character is reversed. See below for clarity.
```
7a3e becomes 3e7a
7c62 becomes 627c
577a becomes 7a57
7725 becomes 2577 
```
Any sane person would ask why though? What is little-endian's purpose? Well the answer is for ease to the computer, let’s say we have a 32 bit computer and currently it’s storing the decimal (base10) value 12.  
If we're **NOT** using little-endian (a.k.a using big-endian), the computer would store the value like  0x00000000000000000000000000000012, as you can see, storing things like this isn’t very efficient, instead, little-endian computer just needs to store the value “12000000000000000000000000000000”, ignoring the trailing zeroes and little-endian computer only need to store "12". 

As mentioned above, big-endian computer which doesn’t reverse the storing and store 0x00000000000000000000000000000012 as is.  
It’s worth mentioning that little-endian and big-endian just determine how computer stores a value, if we do a calculation on each computer, the result is still the same as long as the computer knows it uses a certain endian and stays consistent with it, it should be fine. Next thing you might ask is: How do computers with different endianness communicate with each other? As long as the computer tells the other computer what endian they use, the computer can change the way they interpret each other’s information, as you can see endianness is really just reversing a string, a rather trivial task for any modern computer. 

## Final answer

With that in mind, let's reset this debugger and enter “**3e7a 627c 7a57 2577**”. Don’t forget to check the hex input option. 

![Enter hardcoded password with little-endian](/public/microcorruption - sydney/7.png)

Done 😉️

![Done ;)](/public/microcorruption - sydney/8.png)












