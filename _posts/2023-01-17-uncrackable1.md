---
title: Uncrackable1
date: 2023-01-17 11:00:00 +7
categories: [portable-computer]
tags: [ctf, android]
---

Today I want to share my experience playing [Uncrackable1](https://mas.owasp.org/crackmes/) by OWASP MASTG with some limitations I intentionally put.  
This limitation is: solve Uncrackable1 with only a debugger and nothing else! No Frida, Xposed, or similiar tools.  

Also, please note that different decompiler will yield different output so your results may vary.

Lets start!


## Tools I use
1. Android Studio electric eel for debugger and android emulation
2. jadx 1.4.5 for decompiling
3. [Uncrackable1](https://github.com/OWASP/owasp-mastg/raw/master/Crackmes/Android/Level_01/UnCrackable-Level1.apk) 

## Decompiling
To decompile, simply 
```bash
./jadx UnCrackable-Level1.apk
```

## Root detection evasion

I'm emulating a rooted android, so I get this 

![root detected](/public/uncrackable/root-detected.png)

Lets see what is the root detection actually doing under the hood.

To do this, enable developer mode on your android phone, then select Uncrackable1 to <code> Select debug app </code> and enable <code> Wait for debugger </code> feature.

![enable wait for debugger feature](/public/uncrackable/dev-option.png)

After, you enable those feature, you should get this message when you open Uncrackable1 application

![waiting for debugger when launched](/public/uncrackable/waiting-debugger.png)

Then, <code> attach debugger to android phone </code> on Android Studio  
![attach debugger to android phone](/public/uncrackable/attach-debugger.png)

Tick the <code> Show all processes </code> option and pick <code> owasp.mstg.uncrackable1 </code>

![attaching debugger](/public/uncrackable/attaching.png)

You might've noticed that after you attach the debugger, it automatically show us the root detected message again. This is because we haven't told the debugger to stop anywhere on the execution. To tell the IDE to stop, we must set a breakpoint.

After looking at the decompiled code for a while, I chose the function <code> onCreate </code> as the first breakpoint because it has "Root detected" message written on it. So it suggests that it's doing some root detection (unless the developer try to put dead code) 

![1st breakpoint](/public/uncrackable/oncreate.png)

Reading the decompiled code above, shows that it calls a function from a class called <code> c </code>. Opening this class will ACTUALLY show us how the root detection works under the hood.
```java 
public static boolean a() {
  for (String str : System.getenv("PATH").split(":")) {
    if (new File(str, "su").exists()) {
      return true;
    }
  }
  return false;
}
public static boolean b() {
  String str = Build.TAGS;
  return str != null && str.contains("test-keys");
}

public static boolean c() {
  for (String str : new String[]{"/system/app/Superuser.apk", "/system/xbin/daemonsu", "/system/etc/init.d/99SuperSUDaemon", "/system/bin/.ext/.su", "/system/etc/.has_su_daemon", "/system/etc/.installed_su_daemon", "/dev/com.koushikdutta.superuser.daemon/"}) {
    if (new File(str).exists()) {
      return true;
    }
  }
  return false;
}

```
1. The first check, checks if we have a binary called "su" anywhere on our $PATH which, if exists, usually indicates a rooted device  
2. The second check, checks if the BUILD tag is "test-keys" which usually indicates a custom ROMs  
3. The third check, checks if we have a binaries that's famous to root android device. 

In my case, I only need to worry for the first check since I emulate my android device and usually rooted by default. So I won't have a custom ROMs nor binaries to root android devices.

Anyway, Re-attaching the debugger will stop us at onCreate function. First, 
press "Force Step Into" to bring us NOT to the uncrackable code, but, the libraries that are used. We take advantage of this feature to see/modify the output of libraries function. In this case the output we want to see/modify is the $PATH.  
We could follow the getenv libraries execution one-by-one but I found the easiest way to bypass this is to change the <code> split </code> character to any character except colon (:). This will cause the $PATH to not splitted properly.   

![change split character](/public/uncrackable/change-split.png)

After changing the split character, I could resume the rest of the application without getting root detection message. 

![root not detected](/public/uncrackable/root-not.png)


### Root detection evasion Step-by-step

1. Set a breakpoint on onCreate function from MainActivity.java
2. Attach a debugger to android phone
3. Press "Force Step Into" -> "Step Out" -> "Step Into"
4. Change the regex variable to any character except colon (:)

## Finding the secret

Looking through MainActivity.java again, I see <code> verify </code> function. Before "This is the correct secret" message, it calls a function from class <code> a </code>.  
Opening the function shows us the function that verify the string inputted. Set a breakpoint here. 
```
public static boolean a(String str) {
  byte[] bArr;
  byte[] bArr2 = new byte[0];
  try {
    bArr = sg.vantagepoint.a.a.a(b("8d127684cbc37c17616d806cf50473cc"), Base64.decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0));
  } catch (Exception e) {
    Log.d("CodeCheck", "AES error:" + e.getMessage());
    bArr = bArr2;
  }
  return str.equals(new String(bArr));
}
```

It seems like it also obfuscate the encrypted key using a function called <code> b </code>

```java
public static byte[] b(String str) {
  int length = str.length();
  byte[] bArr = new byte[length / 2];
  for (int i = 0; i < length; i += 2) {
    bArr[i / 2] = (byte) ((Character.digit(str.charAt(i), 16) << 4) + Character.digit(str.charAt(i + 1), 16));
  }
  return bArr;
}
```
So decrypting it probably won't work. 

After, stepping line-by-line (Warning: it's very long) I finally caught this 

![secret](/public/uncrackable/i-want-to-believe+input.png)

On a library called equals, it compares my input (abcde) with the secret (I want to believe).  

Putting it on the app confirms this!

![done](/public/uncrackable/done.png)

### Finding the Secret Step-by-step
1. Set a breakpoint on a function called <code> a </code> in <code>  sources/sg/vantagepoint/uncrackable1/a.java </code>
2. Attach a debgger to android phone.
3. Press: 
    1. Step into
    2. Step out
    3. Step into
    4. Step out 
    5. Step out
    6. Force step into
    7. Step out
    8. Force step into
4. See that the secret is "I want to believe"

## Done :D




