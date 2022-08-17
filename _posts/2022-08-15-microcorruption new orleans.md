---
title: Microcorruption - New Orleans
date: 2022-08-15 10:00:00 +7
categories: [computer]
tags: [ctf,microcorruption,msp430, assembly]
---

This is my writeup for microcorruption CTF level [New Orleans](https://microcorruption.com/debugger/New%20Orleans)

Usually, the first thing I do is "b main" so let's go ahead and do that. But why did I choose "b main"? Why not just start from the absolute top and work our way down to the end? Well, as the name suggest, it's the main function, thus, the main functionality of the program should be located there, also, 99.9% of the time, programmer would need to call the "main" function in order for the program to work.  
One might wonder, what if we fill the main with a bunch of "fake" code and instead we put the "real" code somewhere else, it's safer that way right? To that I'd say, you could, you absolutely could but I wouldn't say its much safer. Why? In security we know a concept "security by obscurity". What this basically means is "we're secure as long as the criminals don't know our secret", this is true for a lot of things in the world for example, intellectual property (IP), it's inevitable, but in computer security (or security in general), we don't want this. Security by obscurity is saying "lets hope that the criminals don't know our secret". You surely don't want to only hope that your system is secure right? You want the system to be ACTUALLY secure, no matter how many people tries to break it, it doesn't' break. In our case, it doesn't matter where you put your code, if it's bug free then it's bug free. Besides, "hiding" your code to other part of the program would only stall a skilled hacker only for a tiny bit. 

I went to a little tangent there xd, moving on,

![after "b main"](/public/microcorruption - new orleans/1.png)

Right off the bat, we see a few interesting things namely: create\_password and check\_password.  
Let's explore create\_password since we encounter them first, type “b create\_password”

## create_password function

![after "b create_password"](/public/microcorruption - new orleans/2.png)

```
447e:  3f40 0024      mov	#0x2400, r15
4482:  ff40 4f00 0000 mov.b	#0x4f, 0x0(r15)
4488:  ff40 7900 0100 mov.b	#0x79, 0x1(r15)
448e:  ff40 6000 0200 mov.b	#0x60, 0x2(r15)
4494:  ff40 7b00 0300 mov.b	#0x7b, 0x3(r15)
449a:  ff40 4200 0400 mov.b	#0x42, 0x4(r15)
44a0:  ff40 4d00 0500 mov.b	#0x4d, 0x5(r15)
44a6:  ff40 5b00 0600 mov.b	#0x5b, 0x6(r15)
44ac:  cf43 0700      mov.b	#0x0, 0x7(r15)
44b0:  3041           ret
```

So this looks like we are creating a password here (duh), lets translate the hex value to ascii and see what we get 
```
Hex value	ascii  
#0x4f	        O
#0x79	        y
#0x60           `
#0x7b	        {
#0x42	        B
#0x4d	        M
#0x5b	        [
#0x0	        NULL
```
So, if we just assume that the lock hardcoded the password, the password would be “Oy`{BM[”.


## Intentionally inputting the wrong password:
---
Lets not get ahead of ourselves and check the other interesting function check\_password, lets go type “b check\_password”   
And we got interrupted. Psst, you can type the hardcoded password from get_password and it’ll be correct but for the sake of completion, lets just make sure that the lock is checking the hardcoded password. I input “password” to see what happens.  
![Interrupted](/public/microcorruption - new orleans/3.png)

---

![after "b check_password"](/public/microcorruption - new orleans/5.png)

Before I explain in more detail what this function does, I want you to notice that the hardcoded password and the password we input can be seen at Live Memory Dump. This is because when the lock calls the function “create_password”, the lock creates the password at memory address 2400 (see instruction 447e).

44bc:  0e43           clr	r14
This just simply clears the r14 register which we’ll see be used to store the index to access the hardcoded password and inputted password

44be:  0d4f           mov	r15, r13
Makes r13 value equal to r15 (in this case it’s 439c). R13 register will be the variable which stores our inputted password

44c0:  0d5e           add	r14, r13
As I stated earlier (instruction 44bc), r14 will be the index, since we’re on our first iteration and r14 is 0, nothing will change here. 

44c2:  ee9d 0024      cmp.b	@r13, 0x2400(r14)
This will compare the value at memory address r13 (in this case the character “p”) and the character at memory 2400+r14. Since 2400 + r14 (0) = 2400, it compares the character “O” with “p”, this obviously isn’t the same character so the sr (status register) won’t carry a Zero flag. 

44c6:  0520           jnz	$+0xc <check_password+0x16>
Since the Zero flag isn’t set at sr (status register), this instruction will jump “c” amount in hexadecimal which is equal to 12. 
We can predict where this jnz will take us, we’re currently on instruction 44c6 and since this instruction will jump “c” amount, we can write: 44c6 + c = 44d2. 

44c8:  1e53           inc	r14
We skipped this because we intentionally input the wrong password. 

44ca:  3e92           cmp	#0x8, r14
We skipped this because we intentionally input the wrong password.

44cc:  f823           jnz	$-0xe <check_password+0x2>
We skipped this because we intentionally input the wrong password.

44ce:  1f43           mov	#0x1, r15
We skipped this because we intentionally input the wrong password.

44d0:  3041           ret
We skipped this because we intentionally input the wrong password.

44d2:  0f43           clr	r15
This will clear the register r15 which we will see be used to check if we inputted the correct password or not. 

44d4:  3041           ret
Return to the function that calls this function (in this case we return to main)


## Inputting the correct password:
---
Repeat the step above and input “Oy`{BM[” when prompted for password.
Below is an explanation for the check_password function.

44bc:  0e43           clr	r14
This simply clears the r14 register for later use.

44be:  0d4f           mov	r15, r13
This copies the value of r15 to the r13.. In other word, r15 = r13

44c0:  0d5e           add	r14, r13
R14 is used to specify the index to check the inputted password and the actual password

44c2:  ee9d 0024      cmp.b	@r13, 0x2400(r14)
From https://microcorruption.com/public/manual.pdf:
cmp arg1 arg2 → compute arg1 - arg2, set the flags, and discard the result
So , this instruction simply subtract the value at address r13 with memory address 2400+r14.
I’ll take the first iteration as an example, the first character at memory address r13 is “O” and the first character at memory address 2400 + r14 is “O”.. So the it should compute “O” - “O” = 0.
Since it computes 0, the Zero (Z) flag should be activated to the SR (status register).

When you hover to see the SR (status register) you’d also see that the C flag is activated.
Carry (C) flag is activated when the computation produces a carry. What does “produces a carry mean”?  Well, one way to know is to actually do the calculation. 
O in binary is “0101” and -5 in binary is “1011”. So , 
```
01001111 
01001111 -
—-------
100000000
```
As you can see, the first bit is what we called a “Carry bit”. The calculation produced a carry bit. 

44c6:  0520           jnz	$+0xc <check_password+0x16>
Since the Zero (Z) flag is set, the jump won’t trigger as opposed when we intentionally input the wrong password.

44c8:  1e53           inc	r14
Increment the index to check the next character.

44ca:  3e92           cmp	#0x8, r14
Check if we’ve reached the end of the actual password (the hardcoded 8 character password). This will also Zero flag if we’ve iterated for 8 times. It’s worth noting that “Oy`{BM[” is 7 characters + 1 “NULL” character so it’s 8 characters. 

44cc:  f823           jnz	$-0xe <check_password+0x2>
If we haven’t iterated for 8 times, jump to instruction 44be.

44ce:  1f43           mov	#0x1, r15
Change the value of r15 to 1. In other word, r15 = 1.

44d0:  3041           ret
Return to main function.


## Final check:

Returning to the main function, we’ll see:
```
4454:  0f93           tst	r15
4456:  0520           jnz	$+0xc <main+0x2a>
```
Check if r15 is zero. This is the final check which will determine if we’ll unlock the lock or not. 

Since r15 isn’t zero, we jump to:

```
4462:  3f40 2045      mov	#0x4520 "Access Granted!", r15
```

![unlocked](/public/microcorruption - new orleans/6.png)

Done 😉️












