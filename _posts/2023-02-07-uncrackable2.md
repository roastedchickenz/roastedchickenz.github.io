---
title: Uncrackable2
date: 2023-02-07 11:00:00 +7
categories: [portable-computer]
tags: [ctf, android]
---

Today I want to share my experience playing [Uncrackable2](https://mas.owasp.org/crackmes/) by OWASP MASTG.

Lets start!


## Tools I use
1. Android Studio electric eel for debugger and android emulation
2. jadx 1.4.5 for decompiling
3. [Uncrackable1](https://github.com/OWASP/owasp-mastg/raw/master/Crackmes/Android/Level_01/UnCrackable-Level1.apk) 

## Decompiling
To decompile, simply 
```bash
./jadx UnCrackable-Level2.apk
```

## Root detection evasion

I'm emulating a rooted android, so I get this 

![root detected](/public/uncrackable2/root-detected.png)

Looking at the decompiled code, the root detection mechanisms looks exactly the same as Uncrackable1, except when I tried to attach a debugger, it detects that I'm on a rooted device. 


So I need to find a new way to bypass the root detection. Finally, I use Frida to bypass the root detection mechanism. 
After starting Frida on my Android emulator, always checks that it has successfully connected to host machine using:
```bash
frida-ps -U
```

Anyway, this is the Frida script that I use

```javascript
Java.perform(function(){
	var root-check-class = Java.use("sg.vantagepoint.a.b");
	root-check-class.a.implementation = function(f){
		return false;
	}
	root-check-class.b.implementation = function(f){
		return false;
	}
	root-check-class.c.implementation = function(f){
		return false;
	}
	console.log("Changes completed");
})
```
Execute the script using:
```bash
frida -U -l uncrackable2-disable-root.js -f owasp.mstg.uncrackable2
```

With that, I successfully evade Uncrackable2's root detection mechanisms!

![root detection evaded](/public/uncrackable2/root-not.png)

## Finding the secret 

Unlike the previous challenge, Uncrackable2's verify is a bit different this time. When we dig through the "verify" function, we found that it calls a native function shown by these lines of code
```java
// sources/sg/vantagepoint/uncrackable2/MainActivity.java

static {
  System.loadLibrary("foo");
}

```

```java
sources/sg/vantagepoint/uncrackable2/CodeCheck.java

public class CodeCheck {
  private native boolean bar(byte[] bArr);

  public boolean a(String str) {
    return bar(str.getBytes());
  }
}
```

We need to dig through the native function this time. First, we need to find the file of course! It should be on the default location but just in case, we can always use something like
```bash
find <path-to-decompiled-apk> | grep -i foo
```
Where "foo" is the native library we want to find as shown by MainActivity.java.  
We should see that it's located on <code> ./resources/lib/x86_64/libfoo.so </code>

The architecture doesn't really matter in this case, I pick x86_64 not for any spesific reason.

When decompiling binaries, I like to start with the easiest thing, giving the binaries to <code> strings </code> command. 
```bash
strings -n 6 libfoo.so
```

In this case, the strings command doesn't give any useful information (Unless you already know the solution, it gives truncated version of the secret string).
Next, try ghidra. 

After auto-analyzing with ghidra, the symbol tree will show us existing function

![symbol tree](/public/uncrackable2/symbol-tree.png)

Reading the decompiled function <code> Java_sg_vantagepoint_uncrackable2_CodeCheck_bar </code>, we could find that it is indeed doing a code check. It uses <code> strncmp </code> function. 

Which means we should be able to retrieve the compared strings.

![decompiled function](/public/uncrackable2/decompiled-function.png)

On line 31, where it does the strncmp function, it points to the value of local_30. 

The value of local_30 is a hex as shown on line 21. In fact, the rest of the secret is laid here in Little-endian and then reversed.  
This is the hex from bottom to top with all the 0x removed 

```text
6873696620656874206c6c6120726f6620736b6e616854
```
The reversed text is <code> Thanks for all the fish </code> 


![done](/public/uncrackable2/done.png)












